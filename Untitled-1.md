*&---------------------------------------------------------------------*
*& Report Z_IOT_POST_PROCESSING
*&---------------------------------------------------------------------*
*& Mô tả: Chương trình hậu kỳ xử lý dữ liệu chấm công từ bảng đệm
*&        ZTA_STAGING sang bảng chính ZTA_TIMESHEET.
*&        - CHUẨN HÓA MENTOR: Đưa toàn bộ khai báo DATA lên đầu.
*&        - TỐI ƯU HIỆU NĂNG: FOR ALL ENTRIES (Bulk Read) & Không Hardcode PLANT.
*&        - MÚI GIỜ VIỆT NAM: Ép chuẩn múi giờ UTC+7 cho ngày giờ quẹt thẻ.
*&        - TÁCH CA ĐÊM & CA MỚI: Khống chế trần time_out ca đêm để không bị đè ca ngày mới.
*&        - UPSERT CHECK-OUT: Chấp nhận quẹt thẻ nhiều lần (First In - Last Out).
*&        - NGÀY LỄ / NGHỈ: CHỈ TÍNH CHỦ NHẬT (DAY = '7') VÀ BẢNG ZTA_HOLIDAY.
*&        - KHỐNG CHẾ TRẦN OT NGÀY LỄ: TOT_HOURS tính thực tế, OT_HOURS ép trần STD_HOURS/OT_PLAN.
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
DATA: lt_staging        TYPE TABLE OF zta_staging,
      ls_stage          TYPE zta_staging,
      lt_timesheet_all  TYPE TABLE OF zta_timesheet,
      ls_timesheet_o    TYPE zta_timesheet,
      ls_timesheet_n    TYPE zta_timesheet,
      lv_card_date      TYPE dats,
      lv_prev_date      TYPE dats,
      lv_card_time      TYPE tims,
      lv_pernr          TYPE pernr_d,
      lv_dept_id        TYPE string,

      " Internal Tables phục vụ tối ưu hóa hiệu năng
      lt_emp_master     TYPE TABLE OF ty_employee,
      ls_emp_master     TYPE ty_employee,
      lt_emp_shifts     TYPE TABLE OF zta_emp_shift,
      ls_emp_shift      TYPE zta_emp_shift,
      lt_schedule       TYPE TABLE OF zta_schedule,
      ls_schedule       TYPE zta_schedule,
      lt_ot_plan        TYPE TABLE OF zta_ot_plan,
      ls_ot_plan        TYPE zta_ot_plan,
      lt_holiday        TYPE TABLE OF zta_holiday,

      " Biến logic định tuyến ca và tính toán thời gian
      lv_matched_shift  TYPE zde_shift_id,
      lv_matched_date   TYPE dats,
      lv_open_shift     TYPE zde_shift_id,
      lv_min_diff       TYPE i,
      lv_diff_in        TYPE i,
      lv_diff_out       TYPE i,
      lv_grace_seconds  TYPE i,
      lv_max_allowed_ot TYPE p DECIMALS 2,
      lv_calculated_ot  TYPE p DECIMALS 2,
      lv_max_seq        TYPE zta_timesheet-seq_no,
      lv_ot_trigger     TYPE c LENGTH 1,

      " Biến chuyên cần và xử lý chuỗi
      lv_day_of_week    TYPE c,
      lv_is_sunday      TYPE c LENGTH 1,
      lv_is_late        TYPE c LENGTH 1,
      lv_is_early       TYPE c LENGTH 1,
      lv_clean_card     TYPE zta_employee-card_id,

      " Biến phục vụ tính toán toán học giờ công & khấu trừ
      lv_sec_dec        TYPE p DECIMALS 2,
      lv_hours          TYPE p DECIMALS 2,
      lv_late_seconds   TYPE i,
      lv_early_seconds  TYPE i,
      lv_deduct_hours   TYPE p DECIMALS 2.

" =========================================================================
" LOGIC XỬ LÝ CHÍNH VÀ ĐỒNG BỘ DỮ LIỆU
" =========================================================================
START-OF-SELECTION.

  " 1. LẤY DỮ LIỆU THÔ từ bảng đệm (Processed = trống HOẶC = 'E' để retry)
  SELECT * FROM zta_staging
    WHERE processed = ' ' OR processed = 'E'
    ORDER BY timestamp
    INTO TABLE @lt_staging.

  IF lt_staging IS INITIAL.
    RETURN.
  ENDIF.

  " Tối ưu RAM: Chuẩn bị trước toàn bộ master data
  LOOP AT lt_staging INTO ls_stage.
    ls_emp_master-card_id = ls_stage-card_id.
    CONDENSE ls_emp_master-card_id NO-GAPS.
    APPEND ls_emp_master TO lt_emp_master.
  ENDLOOP.
  SORT lt_emp_master BY card_id.
  DELETE ADJACENT DUPLICATES FROM lt_emp_master COMPARING card_id.

  " Nạp Master nhân sự
  IF lt_emp_master IS NOT INITIAL.
    SELECT card_id, pernr, dept_id
      FROM zta_employee
      FOR ALL ENTRIES IN @lt_emp_master
      WHERE card_id = @lt_emp_master-card_id
      INTO TABLE @lt_emp_master.
  ENDIF.

  " Nạp tất cả cấu hình ca
  SELECT * FROM zta_schedule INTO TABLE @lt_schedule.

  " Nạp danh mục ngày lễ
  SELECT * FROM zta_holiday INTO TABLE @lt_holiday.

  " 2. VÒNG LẶP XỬ LÝ CHÍNH
  LOOP AT lt_staging INTO ls_stage.

    " A. DỊCH MÚI GIỜ CHUẨN VIỆT NAM (UTC+7)
    CONVERT TIME STAMP ls_stage-timestamp TIME ZONE 'UTC+7'
            INTO DATE lv_card_date TIME lv_card_time.

    IF lv_card_date IS INITIAL.
      ls_stage-processed = 'E'.
      MODIFY zta_staging FROM ls_stage.
      CONTINUE.
    ENDIF.

    lv_prev_date = lv_card_date - 1.

    " B & C. ĐỊNH DANH NHÂN VIÊN TỪ RAM INTERNAL TABLE
    lv_clean_card = ls_stage-card_id.
    CONDENSE lv_clean_card NO-GAPS.

    CLEAR: lv_pernr, lv_dept_id.
    READ TABLE lt_emp_master INTO ls_emp_master WITH KEY card_id = lv_clean_card.
    IF sy-subrc = 0 AND ls_emp_master-pernr IS NOT INITIAL.
      lv_pernr   = ls_emp_master-pernr.
      lv_dept_id = ls_emp_master-dept_id.
    ELSE.
      ls_stage-processed = 'E'.
      MODIFY zta_staging FROM ls_stage.
      CONTINUE.
    ENDIF.

    " D & E. ĐỊNH TUYẾN CA ĐỘNG
    CLEAR: lv_matched_shift, lv_matched_date, ls_schedule.
    lv_min_diff = 999999.

    CLEAR lt_emp_shifts.
    SELECT * FROM zta_emp_shift
      WHERE pernr = @lv_pernr
        AND ( work_date = @lv_card_date OR work_date = @lv_prev_date )
      INTO TABLE @lt_emp_shifts.

    CLEAR lt_timesheet_all.
    SELECT * FROM zta_timesheet
      WHERE pernr = @lv_pernr
        AND ( work_date = @lv_card_date OR work_date = @lv_prev_date )
      INTO TABLE @lt_timesheet_all.

    CLEAR lt_ot_plan.
    SELECT * FROM zta_ot_plan
      WHERE pernr = @lv_pernr
        AND ( plan_date = @lv_card_date OR plan_date = @lv_prev_date )
      INTO TABLE @lt_ot_plan.

    LOOP AT lt_emp_shifts INTO ls_emp_shift.
      READ TABLE lt_schedule INTO ls_schedule WITH KEY shift_id = ls_emp_shift-shift_id.
      IF sy-subrc <> 0. CONTINUE. ENDIF.

      " UPSERT: Kiểm tra ca đang mở Check-in
      CLEAR lv_open_shift.
      READ TABLE lt_timesheet_all INTO ls_timesheet_o
        WITH KEY work_date = ls_emp_shift-work_date
                 shift_id  = ls_emp_shift-shift_id.
      IF sy-subrc = 0 AND ls_timesheet_o-act_in IS NOT INITIAL.
        lv_open_shift = ls_timesheet_o-shift_id.
      ENDIF.

      " --------------------------------------------------------------------
      " ƯU TIÊN 1: XỬ LÝ LƯỢT RA (CHECK-OUT) - KHỐNG CHẾ BIÊN CA ĐÊM
      " --------------------------------------------------------------------
      IF lv_open_shift = ls_emp_shift-shift_id.
        IF ls_schedule-next_day <> 'X' AND ls_emp_shift-work_date = lv_card_date.
          CLEAR lv_diff_out.
          IF lv_card_time >= ls_schedule-time_out.
            lv_diff_out = lv_card_time - ls_schedule-time_out.
          ENDIF.

          IF lv_card_time < ls_schedule-time_out OR lv_diff_out <= 7200.
            lv_matched_shift = ls_emp_shift-shift_id.
            lv_matched_date  = ls_emp_shift-work_date.
            EXIT.
          ENDIF.

        ELSEIF ls_schedule-next_day = 'X'.
          IF ls_emp_shift-work_date = lv_card_date.
            lv_matched_shift = ls_emp_shift-shift_id.
            lv_matched_date  = ls_emp_shift-work_date.
            EXIT.
          ELSEIF ls_emp_shift-work_date = lv_prev_date.
            CLEAR lv_diff_out.
            IF lv_card_time >= ls_schedule-time_out.
              lv_diff_out = lv_card_time - ls_schedule-time_out.
            ENDIF.

            IF lv_card_time < ls_schedule-time_out OR lv_diff_out <= 7200.
              lv_matched_shift = ls_emp_shift-shift_id.
              lv_matched_date  = ls_emp_shift-work_date.
              EXIT.
            ENDIF.
          ENDIF.
        ENDIF.
      ENDIF.

      " --------------------------------------------------------------------
      " ƯU TIÊN 2: XỬ LÝ LƯỢT VÀO (CHECK-IN)
      " --------------------------------------------------------------------
      IF lv_open_shift IS INITIAL AND ls_emp_shift-work_date = lv_card_date.
        IF lv_card_time >= ls_schedule-time_in.
          lv_diff_in = lv_card_time - ls_schedule-time_in.
        ELSE.
          lv_diff_in = ls_schedule-time_in - lv_card_time.
        ENDIF.

        IF lv_diff_in < lv_min_diff.
          lv_min_diff      = lv_diff_in.
          lv_matched_shift = ls_emp_shift-shift_id.
          lv_matched_date  = lv_card_date.
        ENDIF.
      ENDIF.

    ENDLOOP.

    IF lv_matched_shift IS INITIAL.
      ls_stage-processed = 'E'.
      MODIFY zta_staging FROM ls_stage.
      CONTINUE.
    ELSE.
      READ TABLE lt_schedule INTO ls_schedule WITH KEY shift_id = lv_matched_shift.
    ENDIF.

    " F. XÁC ĐỊNH TÍNH CHẤT NGÀY (CHỈ NHẬN CHỦ NHẬT & BẢNG LỄ)
    CLEAR: lv_day_of_week, lv_is_sunday, lv_ot_trigger.

    CALL FUNCTION 'DATE_COMPUTE_DAY'
      EXPORTING date = lv_matched_date
      IMPORTING day  = lv_day_of_week.

    IF lv_day_of_week = '7'. " Chỉ lấy Chủ Nhật (Bỏ Thứ 7 khỏi cờ ngày nghỉ)
      lv_is_sunday = 'X'.
    ENDIF.

    READ TABLE lt_holiday WITH KEY hol_date = lv_matched_date TRANSPORTING NO FIELDS.
    IF sy-subrc = 0 OR lv_is_sunday = 'X'.
      lv_ot_trigger = 'X'.
    ENDIF.

    " G. TRA CỨU TIMESHEET DÒNG CŨ DỰA TRÊN NGÀY CA GỐC
    CLEAR ls_timesheet_o.
    READ TABLE lt_timesheet_all INTO ls_timesheet_o
      WITH KEY pernr = lv_pernr work_date = lv_matched_date shift_id = lv_matched_shift.

    " H. TIẾN HÀNH GHI RECORD CHẤM CÔNG
    IF ls_timesheet_o-act_in IS INITIAL.
      " CHECK-IN
      CLEAR lv_max_seq.
      SELECT MAX( seq_no ) FROM zta_timesheet
        WHERE pernr     = @lv_pernr
          AND work_date = @lv_matched_date
      INTO @lv_max_seq.

      IF lv_max_seq IS INITIAL.
        lv_max_seq = '00'.
      ENDIF.

      CLEAR ls_timesheet_n.
      ls_timesheet_n-mandt     = sy-mandt.
      ls_timesheet_n-pernr     = lv_pernr.
      ls_timesheet_n-work_date = lv_matched_date.
      ls_timesheet_n-seq_no    = lv_max_seq + 1.
      ls_timesheet_n-shift_id  = lv_matched_shift.
      ls_timesheet_n-dept_id   = lv_dept_id.
      ls_timesheet_n-act_in    = lv_card_time.

      lv_grace_seconds = ls_schedule-grace_mins * 60.

      IF lv_ot_trigger = 'X'.
        IF ls_timesheet_n-act_in > ls_schedule-time_in AND
           ( ls_timesheet_n-act_in - ls_schedule-time_in ) > lv_grace_seconds.
          ls_timesheet_n-status  = 'HOLI_LATE'.
        ELSE.
          ls_timesheet_n-status  = 'HOLI_IN'.
        ENDIF.
      ELSE.
        IF ls_timesheet_n-act_in > ls_schedule-time_in AND
           ( ls_timesheet_n-act_in - ls_schedule-time_in ) > lv_grace_seconds.
          ls_timesheet_n-status = 'LATE_IN'.
        ELSE.
          ls_timesheet_n-status = 'CHECK_IN'.
        ENDIF.
      ENDIF.

      INSERT zta_timesheet FROM @ls_timesheet_n.

    ELSE.
      " UPSERT CHECK-OUT
      DELETE FROM zta_timesheet
        WHERE pernr     = @ls_timesheet_o-pernr
          AND work_date = @ls_timesheet_o-work_date
          AND seq_no    = @ls_timesheet_o-seq_no.

      ls_timesheet_n            = ls_timesheet_o.
      ls_timesheet_n-act_out    = lv_card_time.
      ls_timesheet_n-dept_id    = lv_dept_id.

      IF ls_timesheet_n-act_out >= ls_timesheet_n-act_in.
        lv_sec_dec = ls_timesheet_n-act_out - ls_timesheet_n-act_in.
      ELSE.
        lv_sec_dec = ( 86400 - ls_timesheet_n-act_in ) + ls_timesheet_n-act_out.
      ENDIF.
      lv_hours = lv_sec_dec / 3600.

      CLEAR: ls_ot_plan, lv_max_allowed_ot.
      READ TABLE lt_ot_plan INTO ls_ot_plan
        WITH KEY plan_date = lv_matched_date shift_id = lv_matched_shift.
      IF sy-subrc = 0 AND ls_ot_plan-is_ot = 'X'.
        lv_max_allowed_ot = ls_ot_plan-ot_hours.
      ENDIF.

      CLEAR: lv_is_late, lv_is_early.
      lv_grace_seconds = ls_schedule-grace_mins * 60.

      IF ls_timesheet_n-act_in > ls_schedule-time_in AND
         ( ls_timesheet_n-act_in - ls_schedule-time_in ) > lv_grace_seconds.
        lv_is_late = 'X'.
      ENDIF.

      IF ls_timesheet_n-act_out < ls_schedule-time_out.
        lv_is_early = 'X'.
      ENDIF.

      IF lv_ot_trigger = 'X'.
        " TRƯỜNG HỢP: ĐI LÀM NGÀY LỄ / CHỦ NHẬT
        ls_timesheet_n-work_hours = '0.00'.

        IF lv_is_late = 'X' AND lv_is_early = 'X'.
          ls_timesheet_n-status = 'HOLI_LATE'.
        ELSEIF lv_is_late = 'X' AND lv_is_early = ' '.
          ls_timesheet_n-status = 'HOLI_LATE'.
        ELSEIF lv_is_late = ' ' AND lv_is_early = 'X'.
          ls_timesheet_n-status = 'HOLI_EARLY'.
        ELSE.
          ls_timesheet_n-status = 'HOLI_OUT'.
        ENDIF.

        " KHỐNG CHẾ TRẦN OT NGÀY LỄ / CHỦ NHẬT
        CLEAR lv_calculated_ot.
        IF lv_max_allowed_ot > 0.
          lv_calculated_ot = lv_max_allowed_ot.
        ELSE.
          lv_calculated_ot = ls_schedule-std_hours.
        ENDIF.

        IF lv_hours >= lv_calculated_ot.
          ls_timesheet_n-ot_hours = lv_calculated_ot.
        ELSE.
          ls_timesheet_n-ot_hours = lv_hours.
        ENDIF.

      ELSE.
        " TRƯỜNG HỢP: NGÀY LÀM VIỆC BÌNH THƯỜNG TRONG TUẦN (BAO GỒM THỨ 7)
        ls_timesheet_n-ot_hours   = '0.00'.

        IF lv_is_late = 'X' AND lv_is_early = 'X'.
          ls_timesheet_n-status = 'LATE_EARLY'.
        ELSEIF lv_is_late = 'X' AND lv_is_early = ' '.
          IF lv_hours >= ls_schedule-std_hours.
            ls_timesheet_n-status = 'COMPENSATED'.
          ELSE.
            ls_timesheet_n-status = 'LATE_IN'.
          ENDIF.
        ELSEIF lv_is_late = ' ' AND lv_is_early = 'X'.
          ls_timesheet_n-status = 'EARLY_OUT'.
        ELSE.
          ls_timesheet_n-status = 'COMPLETED'.
        ENDIF.

        CLEAR: lv_late_seconds, lv_early_seconds, lv_deduct_hours.

        CASE ls_timesheet_n-status.
          WHEN 'COMPLETED' OR 'COMPENSATED'.
            ls_timesheet_n-work_hours = ls_schedule-std_hours.

            IF lv_hours > ls_schedule-std_hours AND lv_max_allowed_ot > 0.
              lv_calculated_ot = lv_hours - ls_schedule-std_hours.
              IF lv_calculated_ot >= lv_max_allowed_ot.
                ls_timesheet_n-ot_hours = lv_max_allowed_ot.
              ELSE.
                ls_timesheet_n-ot_hours = lv_calculated_ot.
              ENDIF.
            ENDIF.

          WHEN 'LATE_IN'.
            lv_late_seconds = ls_timesheet_n-act_in - ls_schedule-time_in.
            lv_deduct_hours = lv_late_seconds / 3600.
            ls_timesheet_n-work_hours = ls_schedule-std_hours - lv_deduct_hours.

            IF ls_timesheet_n-work_hours > lv_hours.
              ls_timesheet_n-work_hours = lv_hours.
            ENDIF.

          WHEN 'EARLY_OUT'.
            lv_early_seconds = ls_schedule-time_out - ls_timesheet_n-act_out.
            lv_deduct_hours  = lv_early_seconds / 3600.
            ls_timesheet_n-work_hours = ls_schedule-std_hours - lv_deduct_hours.

            IF ls_timesheet_n-work_hours > lv_hours.
              ls_timesheet_n-work_hours = lv_hours.
            ENDIF.

          WHEN 'LATE_EARLY'.
            ls_timesheet_n-work_hours = lv_hours.
        ENDCASE.

        IF ls_timesheet_n-work_hours < 0.
          ls_timesheet_n-work_hours = '0.00'.
        ENDIF.
      ENDIF.

      ls_timesheet_n-tot_hours = lv_hours.
      INSERT zta_timesheet FROM @ls_timesheet_n.
    ENDIF.

    " I. ĐÁNH DẤU HOÀN THÀNH BẢNG ĐỆM
    ls_stage-processed = 'X'.
    MODIFY zta_staging FROM ls_stage.

  ENDLOOP.

  COMMIT WORK.