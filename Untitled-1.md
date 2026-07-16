*&---------------------------------------------------------------------*
*& Report Z_IOT_POST_PROCESSING
*&---------------------------------------------------------------------*
*& Mô tả: Chương trình hậu kỳ xử lý dữ liệu chấm công từ bảng đệm
*&        ZTA_STAGING sang bảng chính ZTA_TIMESHEET.
*&        - CHUẨN HÓA MENTOR: Đưa toàn bộ khai báo DATA lên đầu chương trình.
*&        - HỖ TRỢ ĐA CA TRONG NGÀY (Ca sáng, ca tối độc lập).
*&        - BIÊN NHẬN DIỆN CA ĐỘNG: 5 phút mặc định + Giờ OT phê duyệt.
*&        - HỖ TRỢ CA XUYÊN ĐÊM (NEXT_DAY) & BIÊN TRỄ ÂN HẠN (GRACE_MINS).
*&        - TRẠNG THÁI NGÀY LỄ CHI TIẾT: HOLI_LATE, HOLI_EARLY, HOLI_LATE_EARLY.
*&        - TỐI ƯU HIỆU NĂNG: Sử dụng FOR ALL ENTRIES tối ưu hóa Memory DB.
*&---------------------------------------------------------------------*
REPORT z_iot_post_processing.

TYPES: BEGIN OF ty_employee,
         card_id TYPE zta_employee-card_id,
         pernr   TYPE pernr_d,
         dept_id TYPE string,
       END OF ty_employee.

" =========================================================================
" TOP-DECLARATION BLOCK: ĐẶT TOÀN BỘ KHAI BÁO BIẾN LÊN ĐẦU CHƯƠNG TRÌNH
" =========================================================================
DATA: lt_staging         TYPE TABLE OF zta_staging,
      ls_stage           TYPE zta_staging,
      lt_timesheet_all   TYPE TABLE OF zta_timesheet,
      ls_timesheet_o     TYPE zta_timesheet,
      ls_timesheet_n     TYPE zta_timesheet,
      lv_card_date       TYPE dats,
      lv_prev_date       TYPE dats,
      lv_card_time       TYPE tims,
      lv_pernr           TYPE pernr_d,
      lv_dept_id         TYPE string,

      " Internal Tables phục vụ tối ưu hóa hiệu năng (Bulk Read)
      lt_emp_master      TYPE TABLE OF ty_employee,
      ls_emp_master      TYPE ty_employee,
      lt_emp_shifts      TYPE TABLE OF zta_emp_shift,
      ls_emp_shift       TYPE zta_emp_shift,
      lt_schedule        TYPE TABLE OF zta_schedule,
      ls_schedule        TYPE zta_schedule,
      lt_ot_plan         TYPE TABLE OF zta_ot_plan,
      ls_ot_plan         TYPE zta_ot_plan,
      lt_holiday         TYPE TABLE OF zta_holiday,

      " Biến logic định tuyến ca và tính toán thời gian
      lv_matched_shift   TYPE zde_shift_id,
      lv_matched_date    TYPE dats,
      lv_open_shift      TYPE zde_shift_id,
      lv_min_diff        TYPE i,
      lv_diff_in         TYPE i,
      lv_diff_out        TYPE i,
      lv_grace_seconds   TYPE i,
      lv_max_allowed_ot  TYPE p DECIMALS 2,
      lv_ot_seconds      TYPE i,
      lv_calculated_ot   TYPE p DECIMALS 2,
      lv_max_seq         TYPE zta_timesheet-seq_no,
      lv_ot_trigger      TYPE c LENGTH 1,
      
      " Biến chuyên cần và xử lý chuỗi
      lv_day_of_week     TYPE c,
      lv_is_weekend      TYPE c LENGTH 1,
      lv_is_late         TYPE c LENGTH 1,
      lv_is_early        TYPE c LENGTH 1,
      lv_utc_timestamp   TYPE timestamp,
      lv_clean_card      TYPE zta_employee-card_id,

      " Biến phục vụ tính toán toán học giờ công & khấu trừ
      lv_sec_dec         TYPE p DECIMALS 2,
      lv_hours           TYPE p DECIMALS 2,
      lv_late_seconds    TYPE i,
      lv_early_seconds   TYPE i,
      lv_deduct_hours    TYPE p DECIMALS 2.

" =========================================================================
" LOGIC XỬ LÝ CHÍNH VÀ ĐỒNG BỘ DỮ LIỆU
" =========================================================================
START-OF-SELECTION.

  " 1. LẤY DỮ LIỆU THÔ từ bảng đệm (Processed = trống)
  SELECT * FROM zta_staging
    WHERE processed = ' '
    ORDER BY timestamp
    INTO TABLE @lt_staging.

  IF lt_staging IS INITIAL.
    RETURN.
  ENDIF.

  " TỐI ƯU HIỆU NĂNG (BULK READ): Chuẩn bị trước toàn bộ master data trên RAM
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

  " Nạp tất cả cấu hình ca của Nhà máy 1000
  SELECT * FROM zta_schedule
    WHERE plant = '1000'
    INTO TABLE @lt_schedule.

  " Nạp danh mục ngày lễ
  SELECT * FROM zta_holiday
    WHERE plant = '1000'
    INTO TABLE @lt_holiday.

  " 2. VÒNG LẶP XỬ LÝ CHÍNH
  LOOP AT lt_staging INTO ls_stage.

    " A. DỊCH MÚI GIỜ: Chuyển sang giờ Việt Nam (UTC+7)
    lv_utc_timestamp = ls_stage-timestamp.

    CONVERT TIME STAMP lv_utc_timestamp TIME ZONE 'UTC+7'
            INTO DATE lv_card_date
                 TIME lv_card_time.

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
      READ TABLE lt_schedule INTO ls_schedule WITH KEY plant = '1000' shift_id = ls_emp_shift-shift_id.
      IF sy-subrc <> 0. CONTINUE. ENDIF.

      CLEAR lv_open_shift.
      READ TABLE lt_timesheet_all INTO ls_timesheet_o 
        WITH KEY work_date = ls_emp_shift-work_date 
                 shift_id  = ls_emp_shift-shift_id.
      IF sy-subrc = 0 AND ls_timesheet_o-act_in IS NOT INITIAL AND ls_timesheet_o-act_out = '000000'.
        lv_open_shift = ls_timesheet_o-shift_id.
      ENDIF.

      " --------------------------------------------------------------------
      " ƯU TIÊN 1: XỬ LÝ LƯỢT RA (CHECK-OUT)
      " --------------------------------------------------------------------
      IF lv_open_shift = ls_emp_shift-shift_id.
        IF ( ls_emp_shift-work_date = lv_card_date AND ls_schedule-next_day <> 'X' ) OR
           ( ls_emp_shift-work_date = lv_prev_date AND ls_schedule-next_day = 'X' ).

          IF lv_card_time >= ls_schedule-time_out.
            lv_diff_out = lv_card_time - ls_schedule-time_out.
          ELSE.
            lv_diff_out = ls_schedule-time_out - lv_card_time.
          ENDIF.

          CLEAR lv_max_allowed_ot.
          READ TABLE lt_ot_plan INTO ls_ot_plan 
            WITH KEY plan_date = ls_emp_shift-work_date shift_id = ls_emp_shift-shift_id.
          IF sy-subrc = 0 AND ls_ot_plan-is_ot = 'X'.
            lv_max_allowed_ot = ls_ot_plan-ot_hours.
          ENDIF.
          lv_ot_seconds = lv_max_allowed_ot * 3600.

          IF lv_diff_out <= ( 300 + lv_ot_seconds ).
            lv_matched_shift = ls_emp_shift-shift_id.
            lv_matched_date  = ls_emp_shift-work_date.
            EXIT.
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
      READ TABLE lt_schedule INTO ls_schedule WITH KEY plant = '1000' shift_id = lv_matched_shift.
    ENDIF.

    " F. XÁC ĐỊNH TÍNH CHẤT NGÀY (NGÀY THƯỜNG VS NGÀY NGHỈ)
    CLEAR: lv_day_of_week, lv_is_weekend, lv_ot_trigger.

    CALL FUNCTION 'DATE_COMPUTE_DAY'
      EXPORTING date = lv_matched_date
      IMPORTING day  = lv_day_of_week.

    IF lv_day_of_week = '6' OR lv_day_of_week = '7'.
      lv_is_weekend = 'X'.
    ENDIF.

    READ TABLE lt_holiday WITH KEY plant = '1000' hol_date = lv_matched_date TRANSPORTING NO FIELDS.
    IF sy-subrc = 0 OR lv_is_weekend = 'X'.
      lv_ot_trigger = 'X'.
    ENDIF.

    " G. TRA CỨU TIMESHEET DÒNG CŨ DỰA TRÊN NGÀY CA GỐC
    CLEAR ls_timesheet_o.
    READ TABLE lt_timesheet_all INTO ls_timesheet_o 
      WITH KEY pernr = lv_pernr work_date = lv_matched_date shift_id = lv_matched_shift.

    " H. TIẾN HÀNH GHI RECORD CHẤM CÔNG
    IF ls_timesheet_o-act_in IS INITIAL.
      " ----------------------------------------------------
      " LƯỢT QUẸT ĐẦU TIÊN TRONG CA (CHECK-IN)
      " ----------------------------------------------------
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
      " ----------------------------------------------------
      " CÁC LƯỢT QUẸT TIẾP THEO TRONG CA (GỘP DÒNG CHECK-OUT)
      " ----------------------------------------------------
      DELETE FROM zta_timesheet 
        WHERE pernr     = @ls_timesheet_o-pernr 
          AND work_date = @ls_timesheet_o-work_date 
          AND seq_no    = @ls_timesheet_o-seq_no.

      ls_timesheet_n            = ls_timesheet_o.
      ls_timesheet_n-act_out    = lv_card_time.
      ls_timesheet_n-dept_id    = lv_dept_id.

      " 1. Tính toán số giờ làm việc thực tế
      IF ls_timesheet_n-act_out >= ls_timesheet_n-act_in.
        lv_sec_dec = ls_timesheet_n-act_out - ls_timesheet_n-act_in.
      ELSE.
        lv_sec_dec = ( 86400 - ls_timesheet_n-act_in ) + ls_timesheet_n-act_out.
      ENDIF.
      lv_hours = lv_sec_dec / 3600.

      " 2. Tra cứu phê duyệt kế hoạch tăng ca từ RAM
      CLEAR: ls_ot_plan, lv_max_allowed_ot.
      READ TABLE lt_ot_plan INTO ls_ot_plan 
        WITH KEY plan_date = lv_matched_date shift_id = lv_matched_shift.
      IF sy-subrc = 0 AND ls_ot_plan-is_ot = 'X'.
        lv_max_allowed_ot = ls_ot_plan-ot_hours.
      ENDIF.

      " 3. Đánh giá trạng thái đi muộn / về sớm tổng hợp
      CLEAR: lv_is_late, lv_is_early.
      lv_grace_seconds = ls_schedule-grace_mins * 60.

      IF ls_timesheet_n-act_in > ls_schedule-time_in AND 
         ( ls_timesheet_n-act_in - ls_schedule-time_in ) > lv_grace_seconds.
        lv_is_late = 'X'.
      ENDIF.

      IF ls_timesheet_n-act_out < ls_schedule-time_out.
        lv_is_early = 'X'.
      ENDIF.

      " 4. Rẽ nhánh phân bổ Giờ công và Trạng thái Chuyên cần
      IF lv_ot_trigger = 'X'.
        " TRƯỜNG HỢP: ĐI LÀM NGÀY NGHỈ / LỄ / CUỐI TUẦN
        ls_timesheet_n-work_hours = '0.00'. 

        IF lv_is_late = 'X' AND lv_is_early = 'X'.
          ls_timesheet_n-status = 'HOLI_LATE_EARLY'. 
        ELSEIF lv_is_late = 'X' AND lv_is_early = ' '.
          ls_timesheet_n-status = 'HOLI_LATE'.       
        ELSEIF lv_is_late = ' ' AND lv_is_early = 'X'.
          ls_timesheet_n-status = 'HOLI_EARLY'.      
        ELSE.
          ls_timesheet_n-status = 'HOLI_OUT'.        
        ENDIF.

        IF lv_max_allowed_ot > 0.
          IF lv_hours >= lv_max_allowed_ot.
            ls_timesheet_n-ot_hours = lv_max_allowed_ot.
          ELSE.
            ls_timesheet_n-ot_hours = lv_hours.
          ENDIF.
        ELSE.
          ls_timesheet_n-ot_hours = '0.00'.
        ENDIF.

      ELSE.
        " TRƯỜNG HỢP: NGÀY LÀM VIỆC BÌNH THƯỜNG TRONG TUẦN
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