# -*- coding: utf-8 -*-
import pandas as pd
from flask import Flask, render_template_string, request, redirect, url_for, flash, jsonify
import os
from datetime import datetime, timedelta
import subprocess
import sys
import re
import json
try:
    import unidecode
except ImportError:
    subprocess.check_call([sys.executable, "-m", "pip", "install", "unidecode"])
    import unidecode
app = Flask(__name__)
app.secret_key = 'secret_key_for_flash_messages'
FILE_PATH = 'data.xlsx'
SP_MAPPING_FILE = 'sp_mapping.json'
# Dictionary ánh xạ mã SP sang tên SP và giá mặc định
SP_MAPPING = {
    'SP001': {'name': 'Áo thun nam', 'price': 200000},
    'SP002': {'name': 'Quần jean nữ', 'price': 500000},
    'SP003': {'name': 'Giày thể thao', 'price': 2000000},
    'SP004': {'name': 'Túi xách da', 'price': 1500000},
    'SP005': {'name': 'Đồng hồ đeo tay', 'price': 1000000},
    'SP006': {'name': 'Kính mát thời trang', 'price': 400000},
    'SP007': {'name': 'Ví da bò', 'price': 2000000},
    'SP008': {'name': 'Mũ lưỡi trai', 'price': 200000},
    'SP009': {'name': 'Vớ cotton', 'price': 100000},
    'SP010': {'name': 'Găng tay len', 'price': 300000},
    'SP011': {'name': 'Khăn quàng cổ', 'price': 500000},
    'SP012': {'name': 'Balo du lịch', 'price': 1000000},
    'SP013': {'name': 'Tai nghe bluetooth', 'price': 1000000},
    'SP014': {'name': 'Sạc dự phòng', 'price': 800000},
    'SP015': {'name': 'Ốp lưng điện thoại', 'price': 500000}
}
# -------------------- HÀM CHUẨN HÓA TÊN --------------------
def normalize_text(text):
    return unidecode.unidecode(text).lower()
# -------------------- HÀM TẠO MÃ ĐƠN HÀNG TỰ ĐỘNG --------------------
def get_next_ma_dh(df):
    if df.empty:
        return 'DH001'
    existing = df['MaDH'].astype(str).unique()
    used_numbers = set()
    pattern = re.compile(r'^DH(\d{3})$', re.IGNORECASE)
    for ma in existing:
        match = pattern.match(ma)
        if match:
            used_numbers.add(int(match.group(1)))
    for i in range(1, 1000):
        if i not in used_numbers:
            return f'DH{i:03d}'
    return None
# -------------------- HÀM LẤY GIÁ 
def get_price(ma_sp):
    return SP_MAPPING.get(ma_sp, {}).get('price', 0)
# -------------------- CÁC HÀM PHỤ TRỢ KHÁC --------------------
def load_sp_mapping():
    global SP_MAPPING
    try:
        if os.path.exists(SP_MAPPING_FILE):
            with open(SP_MAPPING_FILE, 'r', encoding='utf-8') as f:
                loaded_data = json.load(f)
                SP_MAPPING = {}
                for key, value in loaded_data.items():
                    if isinstance(value, dict):
                        name = value.get('name', '')
                        price = value.get('price', 0)
                        if isinstance(name, str):
                            name = re.sub(r"[{}'\"\[\]]", '', name).strip()
                        else:
                            name = str(name).strip()
                        if not name:
                            name = f"Sản phẩm {key}"
                        try:
                            price = float(price)
                        except:
                            price = 0
                        SP_MAPPING[key] = {'name': name, 'price': price}
                    else:
                        SP_MAPPING[key] = {'name': f"Sản phẩm {key}", 'price': 0}
                print(f"✅ Đã tải {len(SP_MAPPING)} sản phẩm từ {SP_MAPPING_FILE}")
                check_and_fix_duplicate_sp_names()
    except Exception as e:
        print(f"⚠️ Lỗi tải mapping SP: {e}")
        save_sp_mapping()
def check_and_fix_duplicate_sp_names():
    try:
        duplicate_count = 0
        norm_to_sp = {}
        for ma_sp, info in SP_MAPPING.items():
            norm_name = normalize_text(info['name'])
            if norm_name in norm_to_sp:
                norm_to_sp[norm_name].append(ma_sp)
            else:
                norm_to_sp[norm_name] = [ma_sp]
        for norm_name, sp_codes in norm_to_sp.items():
            if len(sp_codes) > 1:
                print(f"⚠️ Phát hiện {len(sp_codes)} SP trùng tên (không dấu) '{norm_name}': {sp_codes}")
                for i, ma_sp in enumerate(sp_codes):
                    if i > 0:
                        original_name = SP_MAPPING[ma_sp]['name']
                        new_name = f"{original_name} ({ma_sp})"
                        SP_MAPPING[ma_sp]['name'] = new_name
                        duplicate_count += 1
                        print(f"    → Sửa {ma_sp}: '{original_name}' → '{new_name}'")
        if duplicate_count > 0:
            save_sp_mapping()
            df = read_data()
            if not df.empty and 'MaSP' in df.columns:
                for ma_sp, info in SP_MAPPING.items():
                    mask = df['MaSP'].astype(str) == ma_sp
                    if mask.any():
                        df.loc[mask, 'TenSP'] = info['name']
                save_data(df)
                print(f"✅ Đã cập nhật tên SP trong đơn hàng")
    except Exception as e:
        print(f"⚠️ Lỗi khi kiểm tra trùng tên SP: {e}")
def save_sp_mapping():
    try:
        with open(SP_MAPPING_FILE, 'w', encoding='utf-8') as f:
            clean_mapping = {}
            for key, value in SP_MAPPING.items():
                clean_mapping[key] = {
                    'name': str(value.get('name', '')).strip(),
                    'price': float(value.get('price', 0))
                }
            json.dump(clean_mapping, f, ensure_ascii=False, indent=2)
        print(f"✅ Đã lưu {len(SP_MAPPING)} sản phẩm vào {SP_MAPPING_FILE}")
        return True
    except Exception as e:
        print(f"❌ Lỗi lưu mapping SP: {e}")
        return False
def validate_date(date_str, date_format='%d/%m/%Y'):
    try:
        datetime.strptime(date_str, date_format)
        return True
    except ValueError:
        return False
def parse_date(date_str, date_format='%d/%m/%Y'):
    try:
        return datetime.strptime(date_str, date_format)
    except:
        return None
def validate_date_range(ngay_ban_str):
    if not validate_date(ngay_ban_str):
        return False, "Ngày bán không đúng định dạng dd/mm/yyyy"
    ngay_ban = parse_date(ngay_ban_str).date()
    if ngay_ban.year < 2000:
        return False, f"Ngày bán '{ngay_ban_str}' không được trước năm 2000"
    return True, "Ngày hợp lệ"
def format_date_for_display(date_obj):
    try:
        if date_obj is None or pd.isna(date_obj):
            return ''
        if isinstance(date_obj, str):
            try:
                date_obj = datetime.strptime(date_obj, '%d/%m/%Y')
            except:
                try:
                    date_obj = pd.to_datetime(date_obj, dayfirst=True)
                except:
                    return date_obj
        if isinstance(date_obj, (datetime, pd.Timestamp)):
            return date_obj.strftime('%d/%m/%Y')
        return str(date_obj)
    except:
        return str(date_obj)
def read_data():
    if not os.path.exists(FILE_PATH):
        df_init = pd.DataFrame(columns=['TenSP','MaDH','GiaBan','TenKhachHang','SoLuongBan','TongTien','MaSP','NgayBan','TrangThai'])
        df_init.to_excel(FILE_PATH, index=False)
        print(f"✅ Đã tạo file mới: {FILE_PATH}")
    try:
        df = pd.read_excel(FILE_PATH)
        print(f"✅ Đã đọc {len(df)} chi tiết đơn hàng từ {FILE_PATH}")
        required_columns = ['TenSP','MaDH','GiaBan','TenKhachHang','SoLuongBan','TongTien','MaSP','NgayBan','TrangThai']
        for col in required_columns:
            if col not in df.columns:
                df[col] = ''
        if 'TrangThai' in df.columns and df['TrangThai'].isna().any():
            df['TrangThai'].fillna('Đang xử lý', inplace=True)
        if not df.empty and 'TenSP' in df.columns and 'MaSP' in df.columns:
            for idx, row in df.iterrows():
                ma_sp = str(row['MaSP']).strip()
                ten_sp = str(row['TenSP'])
                if pd.notna(ten_sp) and isinstance(ten_sp, str):
                    if re.search(r"\{.*name.*:.*\}|\[.*\]|dict.*|object.*", ten_sp):
                        if ma_sp in SP_MAPPING:
                            df.at[idx, 'TenSP'] = SP_MAPPING[ma_sp]['name']
                        else:
                            df.at[idx, 'TenSP'] = f"Sản phẩm {ma_sp}"
                    elif any(char in ten_sp for char in ['{', '}', '[', ']', "'", '"']):
                        cleaned_name = re.sub(r"[{}'\"\[\]]", '', ten_sp).strip()
                        if cleaned_name:
                            df.at[idx, 'TenSP'] = cleaned_name
    except Exception as e:
        print(f"❌ Lỗi đọc file: {e}")
        df = pd.DataFrame(columns=['TenSP','MaDH','GiaBan','TenKhachHang','SoLuongBan','TongTien','MaSP','NgayBan','TrangThai'])
    if not df.empty and 'NgayBan' in df.columns:
        try:
            df['NgayBan'] = pd.to_datetime(df['NgayBan'], errors='coerce', dayfirst=True)
        except Exception as e:
            print(f"⚠️ Lỗi chuyển đổi ngày: {e}")
    return df
def save_data(df):
    try:
        if 'NgayBan_display' in df.columns:
            df = df.drop(columns=['NgayBan_display'])
        df.to_excel(FILE_PATH, index=False)
        print(f"✅ Đã lưu {len(df)} chi tiết đơn hàng vào {FILE_PATH}")
    except Exception as e:
        print(f"❌ Lỗi lưu file: {e}")
        raise
def group_orders(df):
    if df.empty:
        return pd.DataFrame()
    grouped = df.groupby('MaDH').agg({
        'TenKhachHang': 'first',
        'NgayBan': 'first',
        'TongTien': 'sum',
        'TrangThai': 'first',
        'MaSP': lambda x: list(x),
        'TenSP': lambda x: list(x),
        'GiaBan': lambda x: list(x),
        'SoLuongBan': lambda x: list(x),
    }).reset_index()
    grouped.rename(columns={
        'MaSP': 'DanhSachMaSP',
        'TenSP': 'DanhSachTenSP',
        'GiaBan': 'DanhSachGia',
        'SoLuongBan': 'DanhSachSoLuong'
    }, inplace=True)
    grouped['SoMatHang'] = grouped['DanhSachMaSP'].apply(len)
    return grouped

# -------------------- CÁC ROUTE --------------------
@app.route('/')
def index():
    load_sp_mapping()
    search_keyword = request.args.get('search_keyword', '').strip()
    from_date = request.args.get('from_date', '').strip()
    to_date = request.args.get('to_date', '').strip()
    min_price = request.args.get('min_price', '').strip()
    max_price = request.args.get('max_price', '').strip()
    sort_by = request.args.get('sort_by', 'NgayBan')
    sort_order = request.args.get('sort_order', 'desc')
    # ---- XỬ LÝ NGÀY THÁNG ----
    fd = None
    td = None
    if from_date:
        if validate_date(from_date):
            fd = parse_date(from_date)
        else:
            flash(f'⚠️ Ngày bắt đầu "{from_date}" không đúng định dạng dd/mm/yyyy', 'warning')
            from_date = ''
    if to_date:
        if validate_date(to_date):
            td = parse_date(to_date)
        else:
            flash(f'⚠️ Ngày kết thúc "{to_date}" không đúng định dạng dd/mm/yyyy', 'warning')
            to_date = ''
    # ---- XỬ LÝ GIÁ ----
    filter_min = min_price
    filter_max = max_price
    filter_error = False
    if min_price:
        try:
            min_val = float(min_price)
            if min_val < 1000:
                flash('⚠️ Giá khởi điểm phải từ 1.000 trở lên!', 'warning')
                filter_min = ''
                filter_error = True
        except ValueError:
            flash('⚠️ Giá khởi điểm không hợp lệ!', 'warning')
            filter_min = ''
            filter_error = True
    if max_price:
        try:
            max_val = float(max_price)
            if max_val < 1000:
                flash('⚠️ Giá kết thúc phải từ 1.000 trở lên!', 'warning')
                filter_max = ''
                filter_error = True
        except ValueError:
            flash('⚠️ Giá kết thúc không hợp lệ!', 'warning')
            filter_max = ''
            filter_error = True
    if not filter_error and filter_min and filter_max:
        try:
            if float(filter_min) > float(filter_max):
                flash('⚠️ Giá khởi điểm không thể lớn hơn giá kết thúc!', 'warning')
                filter_min = filter_max = ''
        except ValueError:
            pass
    df = read_data()
    today = datetime.now()
    today_ddmmyyyy = today.strftime('%d/%m/%Y')
    next_ma_dh = get_next_ma_dh(df)
    if next_ma_dh is None:
        flash('⚠️ Đã đạt giới hạn 999 đơn hàng! Không thể thêm mới.', 'warning')
        next_ma_dh = 'HẾT'
    df_grouped = group_orders(df) if not df.empty else pd.DataFrame()
    has_search_criteria = False
    if not df_grouped.empty:
        # Tìm kiếm theo từ khóa
        if search_keyword:
            keyword_norm = normalize_text(search_keyword)
            # Tìm theo mã đơn hàng (chứa)
            mask_madh = df_grouped['MaDH'].astype(str).fillna('').apply(lambda x: keyword_norm in normalize_text(x))
            # Tìm theo tên khách hàng: có từ bắt đầu bằng keyword
            def name_starts_with(name):
                if pd.isna(name):
                    return False
                words = normalize_text(str(name)).split()
                return any(w.startswith(keyword_norm) for w in words)
            mask_tenkh = df_grouped['TenKhachHang'].apply(name_starts_with)
            df_filtered = df_grouped[mask_madh | mask_tenkh].copy()
            if not df_filtered.empty:
                # Tính priority: ưu tiên những tên có từ bắt đầu (0) trước, sau đó đến mã đơn hàng (1)
                df_filtered['priority'] = df_filtered['TenKhachHang'].apply(name_starts_with)
                df_filtered['priority'] = (~df_filtered['priority']).astype(int)  # True -> 0, False -> 1
                df_filtered.sort_values('priority', inplace=True)
                df_filtered.drop(columns='priority', inplace=True)
                df_grouped = df_filtered
                has_search_criteria = True
        # Lọc theo giá
        if filter_min:
            try:
                df_grouped = df_grouped[df_grouped['TongTien'] >= float(filter_min)]
                has_search_criteria = True
            except:
                pass
        if filter_max:
            try:
                df_grouped = df_grouped[df_grouped['TongTien'] <= float(filter_max)]
                has_search_criteria = True
            except:
                pass
        # Lọc theo ngày
        if fd is not None:
            mask = df_grouped['NgayBan'].notna() & (df_grouped['NgayBan'] >= fd)
            df_grouped = df_grouped[mask]
            has_search_criteria = True
        if td is not None:
            td_end = td.replace(hour=23, minute=59, second=59)
            mask = df_grouped['NgayBan'].notna() & (df_grouped['NgayBan'] <= td_end)
            df_grouped = df_grouped[mask]
            has_search_criteria = True
        # Sắp xếp
        if sort_by == 'TrangThaiAsc_MaDHAsc':
            df_grouped['_priority'] = df_grouped['TrangThai'].apply(lambda x: 0 if x == 'Đang xử lý' else 1)
            df_grouped = df_grouped.sort_values(['_priority', 'MaDH'], ascending=[True, True]).drop(columns='_priority')
        elif sort_by == 'TrangThaiAsc_MaDHDesc':
            df_grouped['_priority'] = df_grouped['TrangThai'].apply(lambda x: 0 if x == 'Đang xử lý' else 1)
            df_grouped = df_grouped.sort_values(['_priority', 'MaDH'], ascending=[True, False]).drop(columns='_priority')
        elif sort_by == 'TrangThaiDesc_MaDHAsc':
            df_grouped['_priority'] = df_grouped['TrangThai'].apply(lambda x: 1 if x == 'Đang xử lý' else 0)
            df_grouped = df_grouped.sort_values(['_priority', 'MaDH'], ascending=[True, True]).drop(columns='_priority')
        elif sort_by == 'TrangThaiDesc_MaDHDesc':
            df_grouped['_priority'] = df_grouped['TrangThai'].apply(lambda x: 1 if x == 'Đang xử lý' else 0)
            df_grouped = df_grouped.sort_values(['_priority', 'MaDH'], ascending=[True, False]).drop(columns='_priority')
        elif sort_by == 'TongTien':
            df_grouped = df_grouped.sort_values('TongTien', ascending=(sort_order == 'asc'))
        elif sort_by == 'NgayBan':
            df_grouped = df_grouped.sort_values('NgayBan', ascending=(sort_order == 'asc'))
    total_amount = df_grouped['TongTien'].sum() if not df_grouped.empty else 0
    return render_template_string(HTML_TEMPLATE,
                                 data=df_grouped,
                                 search_keyword=search_keyword,
                                 from_date=from_date,
                                 to_date=to_date,
                                 min_price=min_price,
                                 max_price=max_price,
                                 sort_by=sort_by,
                                 sort_order=sort_order,
                                 today_ddmmyyyy=today_ddmmyyyy,
                                 total_amount=total_amount,
                                 has_search_criteria=has_search_criteria,
                                 format_date_for_display=format_date_for_display,
                                 sp_mapping=SP_MAPPING,
                                 next_ma_dh=next_ma_dh)
@app.route('/add', methods=['POST'])
def add():
    df = read_data()
    try:
        ma_dh = request.form['MaDH'].strip()
        if not re.match(r'^DH\d{3}$', ma_dh):
            flash(f'❌ Mã đơn hàng không hợp lệ!', 'danger')
            return redirect(url_for('index'))
        if not df.empty and ma_dh in df['MaDH'].astype(str).values:
            flash(f'❌ Mã đơn hàng "{ma_dh}" đã tồn tại, vui lòng thử lại!', 'danger')
            return redirect(url_for('index'))
        ten_kh = request.form['TenKhachHang'].strip()
        ngay_ban_str = request.form['NgayBan'].strip()
        valid, msg = validate_date_range(ngay_ban_str)
        if not valid:
            flash(f'❌ {msg}', 'danger')
            return redirect(url_for('index'))
        ngay_ban = parse_date(ngay_ban_str)
        ngay_ban_date = ngay_ban.date()
        items = []
        pattern = re.compile(r'^MaSP_(\d+)$')
        for key, value in request.form.items():
            m = pattern.match(key)
            if m:
                idx = m.group(1)
                ma_sp = value.strip()
                so_luong = float(request.form.get(f'SoLuong_{idx}', 0))
                if ma_sp and ma_sp != 'NEW' and ma_sp in SP_MAPPING:
                    gia = get_price(ma_sp)
                    items.append({
                        'MaSP': ma_sp,
                        'TenSP': SP_MAPPING[ma_sp]['name'],
                        'GiaBan': gia,
                        'SoLuongBan': so_luong,
                        'TongTien': gia * so_luong
                    })
                else:
                    flash(f'❌ Mã SP "{ma_sp}" không hợp lệ', 'danger')
                    return redirect(url_for('index'))
        masp_list = [it['MaSP'] for it in items]
        if len(masp_list) != len(set(masp_list)):
            flash(f'❌ Đơn hàng không thể có các mặt hàng trùng mã SP!', 'danger')
            return redirect(url_for('index'))
        if not items:
            flash('❌ Phải có ít nhất một mặt hàng', 'danger')
            return redirect(url_for('index'))
        new_rows = []
        for it in items:
            new_rows.append({
                'MaDH': ma_dh,
                'TenKhachHang': ten_kh,
                'NgayBan': ngay_ban,
                'TrangThai': 'Đang xử lý',
                **it
            })
        df = pd.concat([df, pd.DataFrame(new_rows)], ignore_index=True)
        save_data(df)
        flash(f'✅ Đã thêm đơn hàng "{ma_dh}" với {len(items)} mặt hàng', 'success')
        return redirect(url_for('index', search_keyword=ma_dh))
    except Exception as e:
        flash(f'❌ Lỗi: {str(e)}', 'danger')
        return redirect(url_for('index'))
@app.route('/edit_order/<ma_dh>', methods=['GET', 'POST'])
def edit_order(ma_dh):
    load_sp_mapping()
    df = read_data()
    if request.method == 'GET':
        items = df[df['MaDH'].astype(str) == str(ma_dh)]
        if items.empty:
            flash('❌ Không tìm thấy đơn hàng', 'danger')
            return redirect(url_for('index'))
        ten_kh = items.iloc[0]['TenKhachHang']
        ngay_ban = format_date_for_display(items.iloc[0]['NgayBan'])
        trang_thai = items.iloc[0]['TrangThai'] if 'TrangThai' in items.columns else 'Đang xử lý'
        items_list = items.to_dict('records')
        cho_phep_sua = (trang_thai == 'Đang xử lý')
        return render_template_string(EDIT_ORDER_TEMPLATE,
                                      ma_dh=ma_dh,
                                      ten_kh=ten_kh,
                                      ngay_ban=ngay_ban,
                                      trang_thai=trang_thai,
                                      items=items_list,
                                      sp_mapping=SP_MAPPING,
                                      cho_phep_sua=cho_phep_sua)
    else:
        try:
            ma_dh_new = request.form['MaDH'].strip()
            if ma_dh_new != ma_dh:
                flash('❌ Không được phép thay đổi mã đơn hàng!', 'danger')
                return redirect(url_for('edit_order', ma_dh=ma_dh))
            trang_thai_moi = request.form['TrangThai'].strip()
            df_old = read_data()
            old_items = df_old[df_old['MaDH'].astype(str) == str(ma_dh)]
            if old_items.empty:
                flash('❌ Đơn hàng không tồn tại', 'danger')
                return redirect(url_for('index'))
            trang_thai_cu = old_items.iloc[0]['TrangThai'] if 'TrangThai' in old_items.columns else 'Đang xử lý'
            cho_phep_sua = (trang_thai_cu == 'Đang xử lý')
            if cho_phep_sua:
                ten_kh = request.form['TenKhachHang'].strip()
                ngay_ban_str = request.form['NgayBan'].strip()
                valid, msg = validate_date_range(ngay_ban_str)
                if not valid:
                    flash(f'❌ {msg}', 'danger')
                    return redirect(url_for('edit_order', ma_dh=ma_dh))
                ngay_ban = parse_date(ngay_ban_str)
                ngay_ban_date = ngay_ban.date()
                items = []
                pattern = re.compile(r'^MaSP_(\d+)$')
                for key, value in request.form.items():
                    m = pattern.match(key)
                    if m:
                        idx = m.group(1)
                        ma_sp = value.strip()
                        so_luong = request.form.get(f'SoLuong_{idx}', '0')
                        try:
                            so_luong = float(so_luong)
                        except:
                            flash('❌ Số lượng không hợp lệ', 'danger')
                            return redirect(url_for('edit_order', ma_dh=ma_dh))
                        if ma_sp and ma_sp != 'NEW' and ma_sp in SP_MAPPING:
                            gia = get_price(ma_sp)  # không dùng ngày
                            items.append({
                                'MaSP': ma_sp,
                                'TenSP': SP_MAPPING[ma_sp]['name'],
                                'GiaBan': gia,
                                'SoLuongBan': so_luong,
                                'TongTien': gia * so_luong
                            })
                        else:
                            flash(f'❌ Mã SP "{ma_sp}" không hợp lệ', 'danger')
                            return redirect(url_for('edit_order', ma_dh=ma_dh))
                masp_list = [it['MaSP'] for it in items]
                if len(masp_list) != len(set(masp_list)):
                    flash('❌ Đơn hàng không thể có các mặt hàng trùng mã SP!', 'danger')
                    return redirect(url_for('edit_order', ma_dh=ma_dh))
                if not items:
                    flash('❌ Phải có ít nhất một mặt hàng', 'danger')
                    return redirect(url_for('edit_order', ma_dh=ma_dh))
            else:
                ten_kh = old_items.iloc[0]['TenKhachHang']
                ngay_ban = old_items.iloc[0]['NgayBan']
                items = []
                for _, row in old_items.iterrows():
                    items.append({
                        'MaSP': row['MaSP'],
                        'TenSP': row['TenSP'],
                        'GiaBan': row['GiaBan'],
                        'SoLuongBan': row['SoLuongBan'],
                        'TongTien': row['TongTien']
                    })
            df = df_old[df_old['MaDH'].astype(str) != str(ma_dh)]
            new_rows = []
            for it in items:
                new_rows.append({
                    'MaDH': ma_dh_new,
                    'TenKhachHang': ten_kh,
                    'NgayBan': ngay_ban,
                    'TrangThai': trang_thai_moi,
                    **it
                })
            df = pd.concat([df, pd.DataFrame(new_rows)], ignore_index=True)
            save_data(df)
            flash(f'✅ Đã cập nhật đơn hàng "{ma_dh_new}"', 'success')
            return redirect(url_for('index', search_keyword=ma_dh_new))
        except Exception as e:
            flash(f'❌ Lỗi: {str(e)}', 'danger')
            return redirect(url_for('edit_order', ma_dh=ma_dh))
@app.route('/delete_order/<ma_dh>')
def delete_order(ma_dh):
    df = read_data()
    if ma_dh in df['MaDH'].astype(str).values:
        df = df[df['MaDH'].astype(str) != str(ma_dh)]
        save_data(df)
        flash(f'✅ Đã xóa đơn hàng "{ma_dh}"', 'success')
    else:
        flash(f'⚠️ Không tìm thấy đơn hàng "{ma_dh}"', 'warning')
    return redirect(url_for('index'))
@app.route('/api/order_details/<ma_dh>')
def api_order_details(ma_dh):
    df = read_data()
    items = df[df['MaDH'].astype(str) == str(ma_dh)]
    if items.empty:
        return jsonify({'success': False, 'error': 'Không tìm thấy'})
    result = []
    for _, row in items.iterrows():
        result.append({
            'ma_sp': row['MaSP'],
            'ten_sp': row['TenSP'],
            'gia': float(row['GiaBan']),
            'so_luong': int(row['SoLuongBan']),
            'thanh_tien': float(row['TongTien'])
        })
    return jsonify({'success': True, 'items': result})
@app.route('/api/add_sp', methods=['POST'])
def api_add_sp():
    try:
        data = request.get_json()
        ma_sp = data.get('ma_sp', '').strip().upper()
        ten_sp = data.get('ten_sp', '').strip()
        gia_sp = data.get('gia_sp', 0)
        if not ma_sp or len(ma_sp) < 2:
            return jsonify({'success': False, 'error': 'Mã SP phải có ít nhất 2 ký tự'})
        if not ten_sp or len(ten_sp) < 2:
            return jsonify({'success': False, 'error': 'Tên SP phải có ít nhất 2 ký tự'})
        try:
            gia_sp = float(gia_sp)
            if gia_sp < 1 or gia_sp > 1000000000:
                return jsonify({'success': False, 'error': 'Giá SP từ 1 đến 1,000,000,000'})
        except:
            return jsonify({'success': False, 'error': 'Giá không hợp lệ'})
        ten_sp_norm = normalize_text(ten_sp)
        for code, info in SP_MAPPING.items():
            if code != ma_sp and normalize_text(info['name']) == ten_sp_norm:
                return jsonify({'success': False, 'error': f'Tên SP "{ten_sp}" đã tồn tại với mã {code}'})
        if ma_sp in SP_MAPPING:
            return jsonify({'success': False, 'error': f'Mã SP "{ma_sp}" đã tồn tại'})
        SP_MAPPING[ma_sp] = {'name': ten_sp, 'price': gia_sp}
        save_sp_mapping()
        return jsonify({'success': True, 'message': f'Đã thêm {ma_sp}'})
    except Exception as e:
        return jsonify({'success': False, 'error': str(e)})
@app.route('/api/delete_sp', methods=['POST'])
def api_delete_sp():
    try:
        data = request.get_json()
        ma_sp = data.get('ma_sp', '').strip().upper()
        if ma_sp not in SP_MAPPING:
            return jsonify({'success': False, 'error': 'Không tồn tại'})
        df = read_data()
        if not df.empty and ma_sp in df['MaSP'].astype(str).values:
            return jsonify({'success': False, 'error': 'SP đang được sử dụng trong đơn hàng'})
        del SP_MAPPING[ma_sp]
        save_sp_mapping()
        return jsonify({'success': True})
    except Exception as e:
        return jsonify({'success': False, 'error': str(e)})
# -------------------- CHẠY ỨNG DỤNG --------------------
if __name__ == '__main__':
    try:
        import unidecode
    except ImportError:
        subprocess.check_call([sys.executable, "-m", "pip", "install", "unidecode", "-q"])
        import unidecode
    load_sp_mapping()
    try:
        from pyngrok import ngrok, conf
        token = input("\n🔑 Nhập token ngrok (Enter để bỏ qua): ").strip()
        if token:
            conf.get_default().auth_token = token
            public_url = ngrok.connect(5000).public_url
            print(f"🌐 LINK CHIA SẺ: {public_url}")
    except Exception as e:
        print(f"⚠️ Không thể khởi tạo ngrok: {e}")
    app.run(host='0.0.0.0', port=5000, debug=False)

