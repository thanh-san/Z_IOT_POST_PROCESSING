*&---------------------------------------------------------------------*
*& Report Z_IOT_POST_PROCESSING
*&---------------------------------------------------------------------*
*& Mô tả: Chương trình hậu kỳ xử lý dữ liệu chấm công từ bảng đệm
*&        ZTA_STAGING sang bảng chính ZTA_TIMESHEET.
*&        - ĐỊNH DANH ĐỘNG NHÂN VIÊN: Tra cứu qua CARD_ID (Chống lỗi NUMC).
*&        - HỖ TRỢ ĐA CA TRONG NGÀY (Ca sáng, ca tối độc lập).
*&        - BIÊN NHẬN DIỆN CA ĐỘNG: 5 phút mặc định + Giờ OT phê duyệt.
*&        - CƠ CHẾ GỘP DÒNG THÔNG MINH & TÍNH CÔNG THEO KHUNG CA CHUẨN.
*&        - ĐỐI CHIẾU HOẠCH ĐỊNH TĂNG CA: Kết hợp bảng ZTA_OT_PLAN.
*&---------------------------------------------------------------------*
REPORT z_iot_post_processing.

DATA: lt_staging        TYPE TABLE OF zta_staging,
      ls_stage          TYPE zta_staging,
      ls_timesheet_o    TYPE zta_timesheet,
      ls_timesheet_n    TYPE zta_timesheet,
      lv_card_date      TYPE dats,
      lv_card_time      TYPE tims,
      lv_pernr          TYPE pernr_d,

      " Biến phục vụ xử lý đa ca và khung giờ biên
      lt_emp_shifts     TYPE TABLE OF zta_emp_shift,
      ls_emp_shift      TYPE zta_emp_shift,
      ls_schedule       TYPE zta_schedule,
      lv_matched_shift  TYPE zde_shift_id,
      lv_allow_in       TYPE tims,
      lv_allow_out      TYPE tims,

      " Biến phục vụ tính toán OT và biên động (Đã gom lên đầu để chống trùng)
      lv_max_allowed_ot TYPE p DECIMALS 2,
      ls_ot_plan_check  TYPE zta_ot_plan,
      lv_ot_seconds     TYPE i,
      ls_ot_plan        TYPE zta_ot_plan,
      lv_calculated_ot  TYPE p DECIMALS 2,
      lv_max_seq        TYPE zta_timesheet-seq_no.

START-OF-SELECTION.

  " 1. LẤY DỮ LIỆU THÔ từ bảng đệm (Processed = trống)
  SELECT * FROM zta_staging
    WHERE processed = ' '
    ORDER BY timestamp
    INTO TABLE @lt_staging.

  IF lt_staging IS INITIAL.
    RETURN.
  ENDIF.

  " 2. VÒNG LẶP XỬ LÝ TỪNG DÒNG DỮ LIỆU THÔ
  LOOP AT lt_staging INTO ls_stage.

    " A. DỊCH MÚI GIỜ: Chuyển sang giờ Việt Nam (UTC+7)
    DATA: lv_utc_timestamp TYPE timestamp.
    lv_utc_timestamp = ls_stage-timestamp.

    CONVERT TIME STAMP lv_utc_timestamp TIME ZONE 'UTC+7'
            INTO DATE lv_card_date
                 TIME lv_card_time.

    " B & C. ĐỊNH DANH ĐỘNG NHÂN VIÊN & PHÒNG BAN TỪ MÃ THẺ HEX IOT
    DATA: lv_raw_card TYPE string,
          lv_dept_id  TYPE string.

    lv_raw_card = ls_stage-card_id.
    CONDENSE lv_raw_card NO-GAPS.

    CLEAR: lv_pernr, lv_dept_id.

    " Truy vấn ngược bảng Master Data dựa vào mã thẻ HEX thu được từ thiết bị
    SELECT SINGLE pernr, dept_id
      FROM zta_employee
      WHERE card_id = @lv_raw_card
      INTO (@lv_pernr, @lv_dept_id).

    " Nếu thẻ này chưa được đăng ký cho bất kỳ nhân sự nào -> Gắn cờ lỗi E và chặn đứng
    IF sy-subrc <> 0 OR lv_pernr IS INITIAL.
      ls_stage-processed = 'E'.
      MODIFY zta_staging FROM ls_stage.
      CONTINUE.
    ENDIF.

    " D & E. ĐỊNH TUYẾN CA ĐỘNG (MỚI: Check-in tự do không giới hạn biên trễ)
    CLEAR: lv_matched_shift, ls_schedule.
    DATA: lv_min_diff   TYPE i VALUE 999999,
          lv_diff_in    TYPE i,
          lv_diff_out   TYPE i,
          lv_open_shift TYPE zde_shift_id.

    " BƯỚC 1: Kiểm tra xem nhân viên có ca nào đang mở (Chưa Check-out) không
    SELECT SINGLE shift_id FROM zta_timesheet
      WHERE pernr     = @lv_pernr
        AND work_date = @lv_card_date
        AND act_in    IS NOT INITIAL
        AND act_out   = '000000'
      INTO @lv_open_shift.

    " BƯỚC 2: Quét danh sách ca phân lịch của nhân viên
    SELECT * FROM zta_emp_shift
      WHERE pernr     = @lv_pernr
        AND work_date = @lv_card_date
      INTO TABLE @lt_emp_shifts.

    IF sy-subrc = 0.
      LOOP AT lt_emp_shifts INTO ls_emp_shift.
        " Tra cứu cấu hình chuẩn của ca
        SELECT SINGLE * FROM zta_schedule
          WHERE plant    = '1000'
            AND shift_id = @ls_emp_shift-shift_id
          INTO @ls_schedule.

        IF sy-subrc = 0.
          " ----------------------------------------------------------------
          " ƯU TIÊN 1: Nếu có ca mở trùng -> Kiểm tra biên Check-out (Giữ biên nghiêm ngặt)
          " ----------------------------------------------------------------
          IF lv_open_shift = ls_emp_shift-shift_id.
            IF lv_card_time >= ls_schedule-time_out.
              lv_diff_out = lv_card_time - ls_schedule-time_out.
            ELSE.
              lv_diff_out = ls_schedule-time_out - lv_card_time.
            ENDIF.

            " Tra cứu kế hoạch OT để tính biên ra năng động
            CLEAR: lv_max_allowed_ot, ls_ot_plan_check, lv_ot_seconds.
            SELECT SINGLE * FROM zta_ot_plan
              WHERE pernr     = @lv_pernr
                AND plan_date = @lv_card_date
                AND shift_id  = @ls_emp_shift-shift_id
              INTO @ls_ot_plan_check.

            IF sy-subrc = 0 AND ls_ot_plan_check-is_ot = 'X'.
              lv_max_allowed_ot = ls_ot_plan_check-ot_hours.
            ENDIF.
            lv_ot_seconds = lv_max_allowed_ot * 3600.

            " Đạt điều kiện biên ra -> Khớp ca để đóng dòng và thoát ngay
            IF lv_diff_out <= ( 300 + lv_ot_seconds ).
              lv_matched_shift = ls_emp_shift-shift_id.
              EXIT.
            ENDIF.
          ENDIF.

          " ----------------------------------------------------------------
          " ƯU TIÊN 2: Nếu không đóng ca mở -> Tính độ lệch Check-in (MỞ RỘNG TỰ DO)
          " ----------------------------------------------------------------
          IF lv_card_time >= ls_schedule-time_in.
            " Đi trễ: Tính khoảng cách từ giờ vào chuẩn đến lúc quẹt thật
            lv_diff_in = lv_card_time - ls_schedule-time_in.
          ELSE.
            " Đi sớm: Tính khoảng cách từ lúc quẹt thật đến giờ vào chuẩn
            lv_diff_in = ls_schedule-time_in - lv_card_time.
          ENDIF.

          " Không check '<= 300' nữa. Hệ thống chỉ tìm ca nào có khoảng cách gần nhất với lượt quẹt
          IF lv_diff_in < lv_min_diff.
            lv_min_diff      = lv_diff_in.
            lv_matched_shift = ls_emp_shift-shift_id.
          ENDIF.

        ENDIF.
      ENDLOOP.
    ENDIF.

    " Kiểm tra kết quả định tuyến cuối cùng
    IF lv_matched_shift IS INITIAL.
      ls_stage-processed = 'E'.
      MODIFY zta_staging FROM ls_stage.
      CONTINUE.
    ELSE.
      " Nạp cấu hình của ca đã khớp
      SELECT SINGLE * FROM zta_schedule
        WHERE plant    = '1000'
          AND shift_id = @lv_matched_shift
        INTO @ls_schedule.
    ENDIF.

    " F. XÁC ĐỊNH NGÀY NGHỈ / LỄ / CUỐI TUẦN ĐỂ TÍNH OT
    DATA: lv_day_of_week TYPE c,
          lv_is_weekend  TYPE c LENGTH 1.
    CLEAR: lv_day_of_week, lv_is_weekend.

    CALL FUNCTION 'DATE_COMPUTE_DAY'
      EXPORTING date = lv_card_date
      IMPORTING day  = lv_day_of_week.

    IF lv_day_of_week = '6' OR lv_day_of_week = '7'.
      lv_is_weekend = 'X'.
    ENDIF.

    DATA: lv_is_holiday TYPE c LENGTH 1.
    CLEAR lv_is_holiday.
    SELECT SINGLE 'X' FROM zta_holiday
      WHERE plant = '1000' AND hol_date = @lv_card_date
      INTO @lv_is_holiday.

    DATA: lv_ot_trigger TYPE c LENGTH 1.
    IF lv_is_holiday = 'X' OR lv_is_weekend = 'X'.
      lv_ot_trigger = 'X'.
    ENDIF.

    " G. TRA CỨU BẢNG CHÍNH TIMESHEET ĐỂ KIỂM TRA DÒNG CŨ
    CLEAR ls_timesheet_o.
    SELECT SINGLE * FROM zta_timesheet
      WHERE pernr     = @lv_pernr
        AND work_date = @lv_card_date
        AND shift_id  = @lv_matched_shift
      INTO @ls_timesheet_o.

    " H. TIẾN HÀNH GHI RECORD CHẤM CÔNG
    IF sy-subrc <> 0.
      " ----------------------------------------------------
      " LƯỢT QUẸT ĐẦU TIÊN TRONG CA (CHECK-IN)
      " ----------------------------------------------------
      CLEAR lv_max_seq.

      SELECT MAX( seq_no ) FROM zta_timesheet
        WHERE pernr     = @lv_pernr
          AND work_date = @lv_card_date
        INTO @lv_max_seq.

      IF lv_max_seq IS INITIAL.
        lv_max_seq = '00'.
      ENDIF.

      CLEAR ls_timesheet_n.
      ls_timesheet_n-mandt     = sy-mandt.
      ls_timesheet_n-pernr     = lv_pernr.
      ls_timesheet_n-work_date = lv_card_date.
      ls_timesheet_n-seq_no    = lv_max_seq + 1.
      ls_timesheet_n-shift_id  = lv_matched_shift.
      ls_timesheet_n-dept_id   = lv_dept_id.
      ls_timesheet_n-act_in    = lv_card_time.

      " Tự động nhận diện đi muộn ngay từ lúc mở ca (Check-in) cho ngày thường
      IF lv_ot_trigger = 'X'.
        ls_timesheet_n-status  = 'HOLI_IN'.
      ELSE.
        IF ls_timesheet_n-act_in > ls_schedule-time_in.
          ls_timesheet_n-status = 'LATE_IN'.
        ELSE.
          ls_timesheet_n-status = 'CHECK_IN'.
        ENDIF.
      ENDIF.

      INSERT zta_timesheet FROM @ls_timesheet_n.

    ELSE.
      " ----------------------------------------------------
      " CÁC LƯỢT QUẸT TIẾP THEO TRONG CA (GỘP DÒNG)
      " ----------------------------------------------------
      DELETE zta_timesheet FROM @ls_timesheet_o.

      ls_timesheet_n            = ls_timesheet_o.
      ls_timesheet_n-act_out    = lv_card_time.
      ls_timesheet_n-dept_id    = lv_dept_id.

      " 1. Tính toán số giờ làm việc thực tế dựa trên mốc quẹt (Dùng làm gốc đối chiếu)
      DATA: lv_sec_dec TYPE p DECIMALS 2,
            lv_hours   TYPE p DECIMALS 2.

      lv_sec_dec = ls_timesheet_n-act_out - ls_timesheet_n-act_in.
      lv_hours   = lv_sec_dec / 3600.

      " 2. Phân bổ giờ công và tra cứu kế hoạch tăng ca động (ZTA_OT_PLAN)
      CLEAR: ls_ot_plan, lv_max_allowed_ot.

      SELECT SINGLE * FROM zta_ot_plan
        WHERE pernr     = @lv_pernr
          AND plan_date = @lv_card_date
          AND shift_id  = @lv_matched_shift
        INTO @ls_ot_plan.

      IF sy-subrc = 0 AND ls_ot_plan-is_ot = 'X'.
        lv_max_allowed_ot = ls_ot_plan-ot_hours. " Giờ OT tối đa được sếp phê duyệt
      ENDIF.

      " Tiến hành phân bổ chi tiết theo tính chất ngày làm việc
      IF lv_ot_trigger = 'X'.
        " ----------------------------------------------------------------
        " TRƯỜNG HỢP: ĐI LÀM NGÀY NGHỈ / LỄ / CUỐI TUẦN -> Tính vào OT hoàn toàn
        " ----------------------------------------------------------------
        ls_timesheet_n-work_hours = '0.00'.
        ls_timesheet_n-status     = 'HOLI_OUT'.

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
        " ----------------------------------------------------------------
        " TRƯỜNG HỢP: NGÀY LÀM VIỆC BÌNH THƯỜNG TRONG TUẦN
        " ----------------------------------------------------------------
        ls_timesheet_n-ot_hours   = '0.00'.

        " Tái phân loại 4 trạng thái chuyên cần động
        DATA: lv_is_late  TYPE c LENGTH 1,
              lv_is_early TYPE c LENGTH 1.
        CLEAR: lv_is_late, lv_is_early.

        IF ls_timesheet_n-act_in > ls_schedule-time_in.
          lv_is_late = 'X'.
        ENDIF.

        IF ls_timesheet_n-act_out < ls_schedule-time_out.
          lv_is_early = 'X'.
        ENDIF.

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

        " Tính Work Hours theo quy chuẩn doanh nghiệp dựa vào Status kết hợp tính toán OT ngày thường
        DATA: lv_late_seconds  TYPE i VALUE 0,
              lv_early_seconds TYPE i VALUE 0,
              lv_deduct_hours  TYPE p DECIMALS 2.

        CASE ls_timesheet_n-status.
          WHEN 'COMPLETED' OR 'COMPENSATED'.
            ls_timesheet_n-work_hours = ls_schedule-std_hours.

            " Tính OT ngày thường: Nếu làm dư thời gian ca chuẩn và có đăng ký kế hoạch tăng ca trước
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

            " CHỐT CHẶN BẢO VỆ: Không cho phép giờ công cao hơn giờ có mặt thực tế
            IF ls_timesheet_n-work_hours > lv_hours.
              ls_timesheet_n-work_hours = lv_hours.
            ENDIF.

          WHEN 'EARLY_OUT'.
            lv_early_seconds = ls_schedule-time_out - ls_timesheet_n-act_out.
            lv_deduct_hours  = lv_early_seconds / 3600.
            ls_timesheet_n-work_hours = ls_schedule-std_hours - lv_deduct_hours.

            " CHỐT CHẶN BẢO VỆ: Tránh lỗi nhảy vọt công khi quẹt ra quá sớm
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

      " Tính toán tổng số giờ công dựa thuần túy trên thời gian quẹt thực tế (Check-out - Check-in)
      ls_timesheet_n-tot_hours = lv_hours.

      INSERT zta_timesheet FROM @ls_timesheet_n.
    ENDIF.

    " I. ĐÁNH DẤU HOÀN THÀNH BẢNG ĐỆM
    ls_stage-processed = 'X'.
    MODIFY zta_staging FROM ls_stage.

  ENDLOOP.

  COMMIT WORK.