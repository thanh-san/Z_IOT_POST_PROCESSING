*&---------------------------------------------------------------------*
*& Report Z_IOT_POST_PROCESSING
*&---------------------------------------------------------------------*
REPORT z_iot_post_processing.

TYPES: BEGIN OF ty_employee,
         card_id TYPE zta_employee-card_id,
         pernr   TYPE pernr_d,
         dept_id TYPE string,
       END OF ty_employee.

" =========================================================================
" TOP-DECLARATION BLOCK
" =========================================================================
DATA: lt_staging         TYPE TABLE OF zta_staging,
      ls_stage           TYPE zta_staging,
      lt_punch_log       TYPE TABLE OF zta_punch_log,
      ls_punch_log       TYPE zta_punch_log,
      lt_timesheet_all   TYPE TABLE OF zta_timesheet,
      ls_timesheet_o     TYPE zta_timesheet,
      ls_timesheet_n     TYPE zta_timesheet,
      lv_card_date       TYPE dats,
      lv_prev_date       TYPE dats,
      lv_card_time       TYPE tims,
      lv_pernr           TYPE pernr_d,
      lv_dept_id         TYPE string,

      " Master Data Tables
      lt_emp_master      TYPE TABLE OF ty_employee,
      ls_emp_master      TYPE ty_employee,
      lt_emp_shifts      TYPE TABLE OF zta_emp_shift,
      ls_emp_shift       TYPE zta_emp_shift,
      lt_schedule        TYPE TABLE OF zta_schedule,
      ls_schedule        TYPE zta_schedule,
      lt_ot_plan         TYPE TABLE OF zta_ot_plan,
      ls_ot_plan         TYPE zta_ot_plan,
      lt_holidays        TYPE TABLE OF zta_holiday,
      ls_holiday         TYPE zta_holiday,

      " Time Window & Shift Routing Variables
      lv_matched_shift   TYPE zde_shift_id,
      lv_matched_date    TYPE dats,
      lv_grace_seconds   TYPE i,
      lv_earliest_in     TYPE tims,
      lv_earliest_out    TYPE tims,
      lv_cutoff_out      TYPE tims,
      lv_ot_seconds      TYPE i,
      lv_max_allowed_ot  TYPE p DECIMALS 2,
      lv_calculated_ot   TYPE p DECIMALS 2,
      lv_exist_punches   TYPE i,

      " Flags
      lv_is_check_in     TYPE abap_bool,
      lv_is_checkout_win TYPE abap_bool,
      lv_is_late         TYPE c LENGTH 1,
      lv_is_early        TYPE c LENGTH 1,
      lv_is_severe_late  TYPE c LENGTH 1,
      lv_has_ot_plan     TYPE abap_bool,
      lv_is_holiday      TYPE abap_bool,
      lv_clean_card      TYPE zta_employee-card_id,
      lv_allow_log       TYPE abap_bool,
      lv_day_of_week     TYPE c LENGTH 1,

      " Calculation Variables
      lv_sec_dec         TYPE p DECIMALS 2,
      lv_hours           TYPE p DECIMALS 2,
      lv_late_seconds    TYPE i,
      lv_early_seconds   TYPE i,
      lv_deduct_hours    TYPE p DECIMALS 2,
      lv_ts              TYPE timestamp.

" =========================================================================
" LOGIC XỬ LÝ CHÍNH
" =========================================================================

START-OF-SELECTION.

  " 1. LẤY DỮ LIỆU THÔ TỪ BẢNG ĐỆM STAGING
  SELECT * FROM zta_staging
    WHERE processed = ' ' OR processed = 'E'
    ORDER BY timestamp
  INTO TABLE @lt_staging.

  IF lt_staging IS INITIAL.
    RETURN.
  ENDIF.

  " Nạp Master nhân sự vào RAM
  LOOP AT lt_staging INTO ls_stage.
    ls_emp_master-card_id = ls_stage-card_id.
    CONDENSE ls_emp_master-card_id NO-GAPS.
    APPEND ls_emp_master TO lt_emp_master.
  ENDLOOP.
  SORT lt_emp_master BY card_id.
  DELETE ADJACENT DUPLICATES FROM lt_emp_master COMPARING card_id.

  IF lt_emp_master IS NOT INITIAL.
    SELECT card_id, pernr, dept_id
      FROM zta_employee
      FOR ALL ENTRIES IN @lt_emp_master
      WHERE card_id = @lt_emp_master-card_id
    INTO TABLE @lt_emp_master.
  ENDIF.

  " Nạp cấu hình ca & ngày lễ
  SELECT * FROM zta_schedule INTO TABLE @lt_schedule.
  SELECT * FROM zta_holiday INTO TABLE @lt_holidays.

  " 2. VÒNG LẶP XỬ LÝ TỪNG LƯỢT QUẸT THẺ
  LOOP AT lt_staging INTO ls_stage.

    " A. QUY ĐỔI MÚI GIỜ VIỆT NAM (UTC+7)
    CONVERT TIME STAMP ls_stage-timestamp TIME ZONE 'UTC+7'
            INTO DATE lv_card_date TIME lv_card_time.

    IF lv_card_date IS INITIAL.
      ls_stage-processed = 'E'.
      MODIFY zta_staging FROM ls_stage.
      CONTINUE.
    ENDIF.

    lv_prev_date = lv_card_date - 1.

    " B. KHỞI TẠO BẢNG NHẬT KÝ QUẸT THÔ ZTA_PUNCH_LOG
    lv_clean_card = ls_stage-card_id.
    CONDENSE lv_clean_card NO-GAPS.

    CLEAR ls_punch_log.
    ls_punch_log-mandt      = sy-mandt.
    TRY.
        ls_punch_log-log_uuid = cl_uuid_factory=>create_system_uuid( )->create_uuid_c32( ).
      CATCH cx_uuid_error.
        GET TIME STAMP FIELD lv_ts.
        ls_punch_log-log_uuid = |{ lv_clean_card }{ lv_ts }|.
    ENDTRY.

    ls_punch_log-card_id    = lv_clean_card.
    ls_punch_log-punch_date = lv_card_date.
    ls_punch_log-punch_time = lv_card_time.

    IF ls_stage-processed = ' '.
      lv_allow_log = abap_true.
    ELSE.
      lv_allow_log = abap_false.
    ENDIF.

    " Kiểm tra nhân viên tồn tại
    CLEAR: lv_pernr, lv_dept_id.
    READ TABLE lt_emp_master INTO ls_emp_master WITH KEY card_id = lv_clean_card.
    IF sy-subrc = 0 AND ls_emp_master-pernr IS NOT INITIAL.
      lv_pernr   = ls_emp_master-pernr.
      lv_dept_id = ls_emp_master-dept_id.
      ls_punch_log-pernr = lv_pernr.
    ELSE.
      IF lv_allow_log = abap_true.
        ls_punch_log-punch_type = 'INVALID'.
        APPEND ls_punch_log TO lt_punch_log.
      ENDIF.
      ls_stage-processed = 'E'.
      MODIFY zta_staging FROM ls_stage.
      CONTINUE.
    ENDIF.

    " C. TẢI LỊCH PHÂN CA VÀ DỮ LIỆU TIMESHEET CỦA NHÂN VIÊN
    CLEAR: lt_emp_shifts, lt_timesheet_all, lt_ot_plan.
    SELECT * FROM zta_emp_shift
      WHERE pernr = @lv_pernr
        AND ( work_date = @lv_card_date OR work_date = @lv_prev_date )
    INTO TABLE @lt_emp_shifts.

    SELECT * FROM zta_timesheet
      WHERE pernr = @lv_pernr
        AND ( work_date = @lv_card_date OR work_date = @lv_prev_date )
    INTO TABLE @lt_timesheet_all.

    SELECT * FROM zta_ot_plan
      WHERE pernr = @lv_pernr
        AND ( plan_date = @lv_card_date OR plan_date = @lv_prev_date )
    INTO TABLE @lt_ot_plan.

    CLEAR: lv_matched_shift, lv_matched_date, ls_schedule.

    " D. ĐỊNH TUYẾN CA & KIỂM TRA CỬA SỔ QUẸT THẺ
    LOOP AT lt_emp_shifts INTO ls_emp_shift.
      READ TABLE lt_schedule INTO ls_schedule WITH KEY shift_id = ls_emp_shift-shift_id.
      IF sy-subrc <> 0. CONTINUE. ENDIF.

      " Mốc vào sớm nhất cho phép: TIME_IN - 1h
      IF ls_schedule-time_in >= 3600.
        lv_earliest_in = ls_schedule-time_in - 3600.
      ELSE.
        lv_earliest_in = '000000'.
      ENDIF.

      " Mốc mở cửa sổ Check-out sớm nhất: TIME_OUT - 1h
      IF ls_schedule-time_out >= 3600.
        lv_earliest_out = ls_schedule-time_out - 3600.
      ELSE.
        lv_earliest_out = '000000'.
      ENDIF.

      " Mốc đóng ca muộn nhất = TIME_OUT + OT + 1h
      CLEAR lv_ot_seconds.
      READ TABLE lt_ot_plan INTO ls_ot_plan
        WITH KEY plan_date = ls_emp_shift-work_date shift_id = ls_emp_shift-shift_id.
      IF sy-subrc = 0 AND ls_ot_plan-is_ot = 'X'.
        lv_ot_seconds = ls_ot_plan-ot_hours * 3600.
      ENDIF.
      lv_cutoff_out = ls_schedule-time_out + lv_ot_seconds + 3600.

      " Kiểm tra khớp ca
      IF ls_schedule-next_day <> 'X' AND ls_emp_shift-work_date = lv_card_date.
        IF lv_card_time >= lv_earliest_in AND lv_card_time <= lv_cutoff_out.
          lv_matched_shift = ls_emp_shift-shift_id.
          lv_matched_date  = ls_emp_shift-work_date.
          EXIT.
        ENDIF.

      ELSEIF ls_schedule-next_day = 'X'.
        IF ls_emp_shift-work_date = lv_card_date.
          IF lv_card_time >= lv_earliest_in.
            lv_matched_shift = ls_emp_shift-shift_id.
            lv_matched_date  = ls_emp_shift-work_date.
            EXIT.
          ENDIF.
        ELSEIF ls_emp_shift-work_date = lv_prev_date.
          IF lv_card_time <= lv_cutoff_out.
            lv_matched_shift = ls_emp_shift-shift_id.
            lv_matched_date  = ls_emp_shift-work_date.
            EXIT.
          ENDIF.
        ENDIF.
      ENDIF.
    ENDLOOP.

    IF lv_matched_shift IS INITIAL.
      IF lv_allow_log = abap_true.
        ls_punch_log-shift_id   = 'NO_SHIFT'.
        ls_punch_log-punch_type = 'NO_SHIFT'.
        APPEND ls_punch_log TO lt_punch_log.
      ENDIF.

      ls_stage-processed = 'E'.
      MODIFY zta_staging FROM ls_stage.
      CONTINUE.
    ENDIF.

    " Đã khớp ca
    READ TABLE lt_schedule INTO ls_schedule WITH KEY shift_id = lv_matched_shift.
    ls_punch_log-shift_id = lv_matched_shift.

    CLEAR ls_timesheet_o.
    READ TABLE lt_timesheet_all INTO ls_timesheet_o
      WITH KEY pernr = lv_pernr work_date = lv_matched_date shift_id = lv_matched_shift.

    " Kiểm tra kế hoạch OT
    CLEAR: ls_ot_plan, lv_has_ot_plan, lv_max_allowed_ot.
    READ TABLE lt_ot_plan INTO ls_ot_plan
      WITH KEY plan_date = lv_matched_date shift_id = lv_matched_shift.
    IF sy-subrc = 0 AND ls_ot_plan-is_ot = 'X'.
      lv_has_ot_plan    = abap_true.
      lv_max_allowed_ot = ls_ot_plan-ot_hours.
    ENDIF.

    " =========================================================================
    " BƯỚC 1: GHI NHẬT KÝ ZTA_PUNCH_LOG (PHÂN LOẠI CHECK_IN/OUT & OT_IN/OUT)
    " =========================================================================
    IF lv_allow_log = abap_true.
      CLEAR lv_exist_punches.

      SELECT COUNT(*) FROM zta_punch_log
        WHERE pernr      = @lv_pernr
          AND shift_id   = @lv_matched_shift
          AND punch_date = @lv_card_date
          AND punch_type <> 'INVALID' AND punch_type <> 'NO_SHIFT'
      INTO @lv_exist_punches.

      LOOP AT lt_punch_log TRANSPORTING NO FIELDS
        WHERE pernr      = lv_pernr
          AND shift_id   = lv_matched_shift
          AND punch_date = lv_card_date
          AND punch_type <> 'INVALID' AND punch_type <> 'NO_SHIFT'.
        lv_exist_punches = lv_exist_punches + 1.
      ENDLOOP.

      lv_exist_punches = lv_exist_punches + 1.

      " Quẹt từ giờ TIME_OUT trở đi khi có lịch OT -> Gán OT_IN / OT_OUT
      IF lv_card_time >= ls_schedule-time_out AND lv_has_ot_plan = abap_true.
        IF ( lv_exist_punches MOD 2 ) = 0.
          ls_punch_log-punch_type = 'OT_OUT'.
        ELSE.
          ls_punch_log-punch_type = 'OT_IN'.
        ENDIF.
      ELSE.
        " Trước TIME_OUT -> Gán CHECK_IN / CHECK_OUT
        IF ( lv_exist_punches MOD 2 ) = 0.
          ls_punch_log-punch_type = 'CHECK_IN'.
        ELSE.
          ls_punch_log-punch_type = 'CHECK_OUT'.
        ENDIF.
      ENDIF.

      APPEND ls_punch_log TO lt_punch_log.
    ENDIF.

    " =========================================================================
    " BƯỚC 2: XỬ LÝ GHI BẢNG TỔNG HỢP CÔNG ZTA_TIMESHEET
    " =========================================================================
    CLEAR: lv_is_check_in, lv_is_checkout_win.

    " 1. Xác định Check-in
    IF ls_timesheet_o-act_in IS INITIAL.
      lv_is_check_in = abap_true.
    ELSE.
      IF ls_schedule-next_day = 'X'.
        IF lv_card_date = ls_timesheet_o-work_date AND lv_card_time < ls_schedule-time_in.
          lv_is_check_in = abap_true.
        ENDIF.
      ELSE.
        IF lv_card_time < ls_schedule-time_in.
          lv_is_check_in = abap_true.
        ENDIF.
      ENDIF.
    ENDIF.

    " 2. Xác định cửa sổ Check-out hợp lệ (từ TIME_OUT - 1h đến TIME_OUT + OT + 1h)
    IF ls_schedule-next_day = 'X'.
      IF lv_card_date > ls_timesheet_o-work_date AND lv_card_time >= lv_earliest_out.
        lv_is_checkout_win = abap_true.
      ENDIF.
    ELSE.
      IF lv_card_time >= lv_earliest_out.
        lv_is_checkout_win = abap_true.
      ENDIF.
    ENDIF.

    lv_grace_seconds = ls_schedule-grace_mins * 60.

    " -------------------------------------------------------------------------
    " NHÁNH 1: CHECK-IN LẦN ĐẦU / QUẸT LẠI TRƯỚC GIỜ VÀO CA
    " -------------------------------------------------------------------------
    IF lv_is_check_in = abap_true.
      CLEAR ls_timesheet_n.
      ls_timesheet_n-mandt     = sy-mandt.
      ls_timesheet_n-pernr     = lv_pernr.
      ls_timesheet_n-work_date = lv_matched_date.
      ls_timesheet_n-shift_id  = lv_matched_shift.
      ls_timesheet_n-dept_id   = lv_dept_id.
      ls_timesheet_n-act_in    = lv_card_time.
      ls_timesheet_n-act_out   = '000000'.

      ls_timesheet_n-seq_no = lv_exist_punches.

      IF ls_timesheet_n-act_in <= ls_schedule-time_in.
        ls_timesheet_n-status = 'CHECK_IN'.
      ELSE.
        lv_late_seconds = ls_timesheet_n-act_in - ls_schedule-time_in.
        IF lv_late_seconds <= lv_grace_seconds.
          ls_timesheet_n-status = 'CHECK_IN'.
        ELSEIF lv_late_seconds <= 3600.
          ls_timesheet_n-status = 'LATE_IN'.
        ELSE.
          ls_timesheet_n-status = 'WARNING'.
        ENDIF.
      ENDIF.

      MODIFY zta_timesheet FROM @ls_timesheet_n.

      READ TABLE lt_timesheet_all TRANSPORTING NO FIELDS
        WITH KEY pernr = ls_timesheet_n-pernr work_date = ls_timesheet_n-work_date shift_id = ls_timesheet_n-shift_id.
      IF sy-subrc = 0.
        MODIFY lt_timesheet_all FROM ls_timesheet_n INDEX sy-tabix.
      ELSE.
        APPEND ls_timesheet_n TO lt_timesheet_all.
      ENDIF.

      " -------------------------------------------------------------------------
      " NHÁNH 2: QUẸT GIỮA CA (Trước TIME_OUT - 1h -> Tăng SEQ_NO, KHÔNG gán ACT_OUT)
      " -------------------------------------------------------------------------
    ELSEIF lv_is_checkout_win = abap_false.
      ls_timesheet_n        = ls_timesheet_o.
      ls_timesheet_n-seq_no = lv_exist_punches.
      MODIFY zta_timesheet FROM @ls_timesheet_n.

      READ TABLE lt_timesheet_all TRANSPORTING NO FIELDS
        WITH KEY pernr = ls_timesheet_n-pernr work_date = ls_timesheet_n-work_date shift_id = ls_timesheet_n-shift_id.
      IF sy-subrc = 0.
        MODIFY lt_timesheet_all FROM ls_timesheet_n INDEX sy-tabix.
      ENDIF.

      " -------------------------------------------------------------------------
      " NHÁNH 3: CHECK-OUT HỢP LỆ (Từ TIME_OUT - 1h đến đóng ca -> CẬP NHẬT ĐÈ ACT_OUT)
      " -------------------------------------------------------------------------
    ELSE.
      ls_timesheet_n            = ls_timesheet_o.
      ls_timesheet_n-seq_no     = lv_exist_punches.
      ls_timesheet_n-act_in     = ls_timesheet_o-act_in. " Giữ nguyên giờ vào ban đầu
      ls_timesheet_n-act_out    = lv_card_time.          " Luôn cập nhật giờ ra mới nhất
      ls_timesheet_n-dept_id    = lv_dept_id.

      " 1. Tính tổng thời gian thực tế có mặt (TOT_HOURS = ACT_OUT - ACT_IN)
      IF ls_timesheet_n-act_out >= ls_timesheet_n-act_in.
        lv_sec_dec = ls_timesheet_n-act_out - ls_timesheet_n-act_in.
      ELSE.
        lv_sec_dec = ( 86400 - ls_timesheet_n-act_in ) + ls_timesheet_n-act_out.
      ENDIF.
      lv_hours = lv_sec_dec / 3600.
      ls_timesheet_n-tot_hours = lv_hours.

      " 2. Đánh giá cờ vi phạm
      CLEAR: lv_is_late, lv_is_early, lv_is_severe_late.

      IF ls_timesheet_n-act_in > ls_schedule-time_in.
        lv_late_seconds = ls_timesheet_n-act_in - ls_schedule-time_in.
        IF lv_late_seconds > 3600.
          lv_is_severe_late = 'X'.
        ELSEIF lv_late_seconds > lv_grace_seconds.
          lv_is_late = 'X'.
        ENDIF.
      ENDIF.

      IF ls_schedule-next_day = 'X'.
        IF lv_card_date = ls_timesheet_n-work_date.
          lv_is_early = 'X'.
        ELSEIF lv_card_date > ls_timesheet_n-work_date AND ls_timesheet_n-act_out < ls_schedule-time_out.
          lv_is_early = 'X'.
        ENDIF.
      ELSE.
        IF ls_timesheet_n-act_out < ls_schedule-time_out.
          lv_is_early = 'X'.
        ENDIF.
      ENDIF.

      " 3. Phân loại trạng thái & Tính toán WORK_HOURS / OT_HOURS
      IF ls_timesheet_o-status = 'WARNING' OR lv_is_severe_late = 'X'.
        ls_timesheet_n-status     = 'WARNING'.
        ls_timesheet_n-work_hours = ls_schedule-std_hours.
        ls_timesheet_n-ot_hours   = '0.00'.

      ELSEIF lv_is_late = 'X' AND lv_is_early = 'X'.
        ls_timesheet_n-status = 'WARNING'.
        lv_late_seconds  = ls_timesheet_n-act_in - ls_schedule-time_in.

        IF ls_schedule-next_day = 'X' AND lv_card_date = ls_timesheet_n-work_date.
          lv_early_seconds = ( 86400 - ls_timesheet_n-act_out ) + ls_schedule-time_out.
        ELSE.
          lv_early_seconds = ls_schedule-time_out - ls_timesheet_n-act_out.
        ENDIF.

        lv_deduct_hours  = ( lv_late_seconds + lv_early_seconds ) / 3600.
        ls_timesheet_n-work_hours = ls_schedule-std_hours - lv_deduct_hours.
        ls_timesheet_n-ot_hours   = '0.00'.

      ELSEIF lv_is_late = 'X' AND lv_is_early = ' '.
        IF lv_hours >= ls_schedule-std_hours.
          ls_timesheet_n-status     = 'COMPLETED'.
          ls_timesheet_n-work_hours = ls_schedule-std_hours.
          " Tính OT nếu có dôi dư sau khi bù giờ
          IF lv_hours > ls_schedule-std_hours AND lv_max_allowed_ot > 0.
            lv_calculated_ot = lv_hours - ls_schedule-std_hours.
            IF lv_calculated_ot >= lv_max_allowed_ot.
              ls_timesheet_n-ot_hours = lv_max_allowed_ot.
            ELSE.
              ls_timesheet_n-ot_hours = lv_calculated_ot.
            ENDIF.
          ELSE.
            ls_timesheet_n-ot_hours = '0.00'.
          ENDIF.
        ELSE.
          ls_timesheet_n-status = 'LATE_IN'.
          lv_late_seconds = ls_timesheet_n-act_in - ls_schedule-time_in.
          lv_deduct_hours = lv_late_seconds / 3600.
          ls_timesheet_n-work_hours = ls_schedule-std_hours - lv_deduct_hours.
          ls_timesheet_n-ot_hours = '0.00'.
        ENDIF.

      ELSEIF lv_is_late = ' ' AND lv_is_early = 'X'.
        ls_timesheet_n-status = 'EARLY_OUT'.
        IF ls_schedule-next_day = 'X' AND lv_card_date = ls_timesheet_n-work_date.
          lv_early_seconds = ( 86400 - ls_timesheet_n-act_out ) + ls_schedule-time_out.
        ELSE.
          lv_early_seconds = ls_schedule-time_out - ls_timesheet_n-act_out.
        ENDIF.
        lv_deduct_hours  = lv_early_seconds / 3600.
        ls_timesheet_n-work_hours = ls_schedule-std_hours - lv_deduct_hours.
        ls_timesheet_n-ot_hours   = '0.00'.

      ELSE.
        " Làm đủ hoặc dư giờ -> COMPLETED
        ls_timesheet_n-status     = 'COMPLETED'.
        ls_timesheet_n-work_hours = ls_schedule-std_hours.

        " Tự động tính OT khi có kế hoạch và quẹt sau giờ tan ca
        IF lv_hours > ls_schedule-std_hours AND lv_max_allowed_ot > 0.
          lv_calculated_ot = lv_hours - ls_schedule-std_hours.
          IF lv_calculated_ot >= lv_max_allowed_ot.
            ls_timesheet_n-ot_hours = lv_max_allowed_ot.
          ELSE.
            ls_timesheet_n-ot_hours = lv_calculated_ot.
          ENDIF.
        ELSE.
          ls_timesheet_n-ot_hours = '0.00'.
        ENDIF.
      ENDIF.

      " 4. KIỂM TRA CHỦ NHẬT (DAY = 7) HOẶC NGÀY LỄ (ZTA_HOLIDAY)
      CLEAR: lv_day_of_week, lv_is_holiday.

      CALL FUNCTION 'DATE_COMPUTE_DAY'
        EXPORTING
          date = ls_timesheet_n-work_date
        IMPORTING
          day  = lv_day_of_week.

      READ TABLE lt_holidays INTO ls_holiday
        WITH KEY hol_date = ls_timesheet_n-work_date.
      IF sy-subrc = 0.
        lv_is_holiday = abap_true.
      ENDIF.

      IF lv_day_of_week = '7' OR lv_is_holiday = abap_true.
        ls_timesheet_n-ot_hours   = ls_timesheet_n-work_hours + ls_timesheet_n-ot_hours.
        ls_timesheet_n-work_hours = '0.00'.
      ENDIF.

      IF ls_timesheet_n-work_hours < 0.
        ls_timesheet_n-work_hours = '0.00'.
      ENDIF.

      MODIFY zta_timesheet FROM @ls_timesheet_n.

      READ TABLE lt_timesheet_all TRANSPORTING NO FIELDS
        WITH KEY pernr = ls_timesheet_n-pernr work_date = ls_timesheet_n-work_date shift_id = ls_timesheet_n-shift_id.
      IF sy-subrc = 0.
        MODIFY lt_timesheet_all FROM ls_timesheet_n INDEX sy-tabix.
      ENDIF.
    ENDIF.

    " Đánh dấu hoàn tất dòng Staging
    ls_stage-processed = 'X'.
    MODIFY zta_staging FROM ls_stage.

  ENDLOOP.

  " GHI BULK VÀO BẢNG NHẬT KÝ QUẸT THÔ ZTA_PUNCH_LOG
  IF lt_punch_log IS NOT INITIAL.
    INSERT zta_punch_log FROM TABLE @lt_punch_log.
  ENDIF.

  COMMIT WORK.