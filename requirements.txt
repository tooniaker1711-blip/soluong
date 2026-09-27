import os
from datetime import datetime
import pandas as pd
import streamlit as st

# ==============================================================================
# CẤU HÌNH STREAMLIT
# ==============================================================================
try:
    st.image("logo1.jpg") # Giữ nguyên cấu hình hiển thị logo của bạn
except Exception:
    pass

st.set_page_config(
    page_title="Hệ Thống Soạn Đồ Yến Tiệc",
    page_icon="🎉",
    layout="wide"
)

# Đường dẫn file dữ liệu dùng chung trên máy chủ/máy local
CSV_FILE = "banquet_history.csv"

# ==============================================================================
# DANH MỤC VẬT DỤNG YẾN TIỆC
# ==============================================================================
banquet_items = {
    "🧺 LINEN – ĐỒ VẢI": [
        "Table Cloth – Khăn bàn", "Overlay", "Chair Cover – Bao ghế", 
        "Chair Bow – Nơ ghế", "Napkin – Khăn ăn", "Skirting – Khăn quây bàn"
    ],
    "🍽️ TABLEWARE – DỤNG CỤ TRÊN BÀN": [
        "Dinner Plate – Đĩa ăn", "Side Plate – Đĩa bánh mì", "Soup Bowl – Chén/tô súp",
        "Dinner Fork – Nĩa ăn", "Dinner Knife – Dao ăn", "Beef Knife – Dao bò",
        "Soup Spoon – Muỗng súp", "Dessert Spoon – Muỗng tráng miệng", 
        "Water Glass – Ly nước", "Wine Glass – Ly vang", 
        "Serving Spoon – Muỗng phục vụ", "Serving Fork – Nĩa phục vụ"
    ],
    "🪑 TABLE & CHAIR – BÀN GHẾ": [
        "Bàn tròn", "Bàn chữ nhật", "Bàn buffet", "Ghế tiệc", "Ghế dự phòng"
    ],
    "🥤 BEVERAGE – SOẠN NƯỚC": [
        "Nước suối", "Nước ngọt", "Nước trái cây", "Nước có gas",
        "Bình nước", "Ly nước", "Xô đá", "Kẹp đá", "Khay phục vụ"
    ]
}

# ==============================================================================
# SESSION STATE TỰ ĐỘNG TẢI DỮ LIỆU
# ==============================================================================
if "checklist_dict" not in st.session_state:
    st.session_state.checklist_dict = {}

if "history" not in st.session_state:
    if os.path.exists(CSV_FILE):
        try:
            df_loaded = pd.read_csv(CSV_FILE)
            st.session_state.history = df_loaded.to_dict(orient="records")
        except Exception:
            st.session_state.history = []
    else:
        st.session_state.history = []

if "admin_logged_in" not in st.session_state:
    st.session_state.admin_logged_in = False

# ==============================================================================
# SIDEBAR
# ==============================================================================
st.sidebar.title("🎉 QUẢN LÝ YẾN TIỆC")
st.sidebar.info("👤 **Người phụ trách:** 4 nhóm soạn đồ\n\n👥 **Quy mô:** 500 khách")
page = st.sidebar.radio("📋 Chọn trang hệ thống", ["📝 Soạn đồ", "🔑 Admin"])

# ==============================================================================
# TRANG SOẠN ĐỒ
# ==============================================================================
if page == "📝 Soạn đồ":
    st.title("🎉 Hệ thống Soạn đồ Yến tiệc (500 Khách)")
    st.caption("Thêm vật dụng cần thiết và cập nhật tiến độ lưu trữ trực tiếp vào CSV")
    st.markdown("---")
    
    col1, col2 = st.columns([1, 2.5])
    
    # --------------------------------------------------------------
    # CỘT CHỌN VẬT DỤNG
    # --------------------------------------------------------------
    with col1:
        st.subheader("➕ Thêm vật dụng")
        category = st.selectbox("📂 Chọn nhóm", list(banquet_items.keys()))
        item = st.selectbox("🍽️ Chọn vật dụng", banquet_items[category])
        
        quantity_needed = st.number_input("🔢 Số lượng cần soạn", min_value=1, step=1, value=500)
        
        if st.button("📥 Thêm vào checklist", use_container_width=True):
            if item not in st.session_state.checklist_dict:
                st.session_state.checklist_dict[item] = {
                    "Nhóm": category.split(" – ")[0].split(" ", 1)[-1],
                    "Vật dụng": item,
                    "Cần soạn": quantity_needed,
                    "Đã soạn": 0
                }
                st.success(f"Đã thêm {item} vào checklist!")
            else:
                st.session_state.checklist_dict[item]["Cần soạn"] = quantity_needed
                st.success(f"Đã cập nhật số lượng cho {item}!")
            st.rerun()

    # --------------------------------------------------------------
    # CỘT BẢNG CHECKLIST
    # --------------------------------------------------------------
    with col2:
        st.subheader("📋 Bảng Checklist Soạn Đồ")
        if st.session_state.checklist_dict:
            df = pd.DataFrame.from_dict(st.session_state.checklist_dict, orient="index").reset_index(drop=True)
            
            df["Còn thiếu"] = df["Cần soạn"] - df["Đã soạn"]
            df["Còn thiếu"] = df["Còn thiếu"].apply(lambda x: x if x > 0 else 0)
            df["Trạng thái"] = df["Còn thiếu"].apply(lambda x: "✅ Đủ" if x == 0 else "⚠️ Thiếu")
            
            df = df[["Nhóm", "Vật dụng", "Cần soạn", "Đã soạn", "Còn thiếu", "Trạng thái"]]
            
            st.write("💡 *Mẹo: Click đúp vào cột **Đã soạn** để cập nhật số lượng.*")
            
            edited_df = st.data_editor(
                df,
                column_config={
                    "Nhóm": st.column_config.TextColumn("Nhóm", disabled=True),
                    "Vật dụng": st.column_config.TextColumn("Vật dụng", disabled=True),
                    "Cần soạn": st.column_config.NumberColumn("Cần soạn", disabled=True),
                    "Đã soạn": st.column_config.NumberColumn("Đã soạn", min_value=0, step=1),
                    "Còn thiếu": st.column_config.NumberColumn("Còn thiếu", disabled=True),
                    "Trạng thái": st.column_config.TextColumn("Trạng thái", disabled=True),
                },
                hide_index=True,
                use_container_width=True
            )

            # Đồng bộ thay đổi trên giao diện vào session state
            for index, row in edited_df.iterrows():
                item_name = row["Vật dụng"]
                if item_name in st.session_state.checklist_dict:
                    st.session_state.checklist_dict[item_name]["Đã soạn"] = row["Đã soạn"]

            st.markdown("---")
            col_btn1, col_btn2 = st.columns(2)
            
            with col_btn1:
                if st.button("💾 Lưu tiến độ xuống File", use_container_width=True):
                    now_str = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
                    
                    # Cập nhật lịch sử
                    for idx, row in edited_df.iterrows():
                        st.session_state.history.append({
                            "Thời gian cập nhật": now_str,
                            "Nhóm": row["Nhóm"],
                            "Vật dụng": row["Vật dụng"],
                            "Cần soạn": row["Cần soạn"],
                            "Đã soạn": row["Đã soạn"],
                            "Trạng thái": row["Trạng thái"]
                        })
                        
                    # Lưu xuống CSV
                    try:
                        df_history = pd.DataFrame(st.session_state.history)
                        df_history.to_csv(CSV_FILE, index=False, encoding="utf-8-sig")
                        st.success("✅ Lưu dữ liệu thành công vào file CSV!")
                    except Exception as e:
                        st.error(f"Lỗi ghi dữ liệu xuống máy chủ: {e}")
                        
            with col_btn2:
                if st.button("🗑️ Xóa toàn bộ checklist đang xem", use_container_width=True):
                    st.session_state.checklist_dict = {}
                    st.rerun()
        else:
            st.info("📋 Checklist đang trống. Hãy thêm các vật dụng cần chuẩn bị từ bảng bên trái.")

# ==============================================================================
# TRANG ADMIN
# ==============================================================================
elif page == "🔑 Admin":
    st.title("🔑 Trang Quản Trị Yến Tiệc")
    
    if not st.session_state.admin_logged_in:
        with st.form("admin_login_form"):
            password = st.text_input("🔐 Nhập mật khẩu quản trị", type="password")
            login_submitted = st.form_submit_button("🔑 Đăng nhập")
            if login_submitted:
                if password == "123456":
                    st.session_state.admin_logged_in = True
                    st.success("Đăng nhập thành công!")
                    st.rerun()
                else:
                    st.error("❌ Mật khẩu không chính xác!")
        st.stop()

    col_header_title, col_header_btn = st.columns([4, 1])
    with col_header_title:
        st.success("🟢 Đăng nhập thành công. Bạn đang xem dữ liệu hệ thống.")
    with col_header_btn:
        if st.button("🔒 Đăng xuất"):
            st.session_state.admin_logged_in = False
            st.rerun()

    tab1, tab2, tab3 = st.tabs([
        "📋 Danh sách vật dụng",
        "📦 Tình trạng soạn đồ",
        "📊 Thống kê & tổng hợp",
    ])
    
    # TAB 1 - DANH SÁCH VẬT DỤNG GỐC
    with tab1:
        st.subheader("📋 Danh mục vật dụng tiêu chuẩn")
        data = []
        for category_name in banquet_items:
            for item_name in banquet_items[category_name]:
                data.append([category_name, item_name])
        df_items = pd.DataFrame(data, columns=["Nhóm", "Tên vật dụng"])
        st.dataframe(df_items, use_container_width=True, hide_index=True)

    # TAB 2 - TÌNH TRẠNG SOẠN ĐỒ
    with tab2:
        st.subheader("📦 Lịch sử soạn đồ chi tiết (Cập nhật từ CSV)")
        if os.path.exists(CSV_FILE):
            try:
                df_history = pd.read_csv(CSV_FILE)
            except Exception:
                df_history = pd.DataFrame()
        else:
            df_history = pd.DataFrame()
            
        if not df_history.empty:
            # Lọc lấy dòng có thời gian cập nhật mới nhất cho từng vật dụng
            df_latest = df_history.sort_values("Thời gian cập nhật").drop_duplicates(subset=["Vật dụng"], keep="last")
            df_latest.sort_values(by=["Nhóm", "Vật dụng"], inplace=True)
            
            df_latest["Còn thiếu"] = df_latest["Cần soạn"] - df_latest["Đã soạn"]
            df_latest["Còn thiếu"] = df_latest["Còn thiếu"].apply(lambda x: x if x > 0 else 0)
            
            st.dataframe(
                df_latest[["Thời gian cập nhật", "Nhóm", "Vật dụng", "Cần soạn", "Đã soạn", "Còn thiếu", "Trạng thái"]],
                use_container_width=True,
                hide_index=True
            )
        else:
            st.info("Hệ thống chưa có dữ liệu lưu trữ.")

    # TAB 3 - THỐNG KÊ
    with tab3:
        st.subheader("📊 Thống kê tổng hợp tình trạng đồ đạc")
        if os.path.exists(CSV_FILE):
            try:
                df_anal = pd.read_csv(CSV_FILE)
            except Exception:
                df_anal = pd.DataFrame()
        else:
            df_anal = pd.DataFrame()
            
        if not df_anal.empty:
            # Chỉ tính trên dữ liệu cập nhật mới nhất
            df_latest = df_anal.sort_values("Thời gian cập nhật").drop_duplicates(subset=["Vật dụng"], keep="last")
            
            tong_khach = 500
            tong_can_soan = df_latest["Cần soạn"].sum()
            tong_da_soan = df_latest["Đã soạn"].sum()
            
            tien_do = (tong_da_soan / tong_can_soan * 100) if tong_can_soan > 0 else 0.0
            con_thieu = tong_can_soan - tong_da_soan if (tong_can_soan - tong_da_soan) > 0 else 0
            
            col_kpi1, col_kpi2, col_kpi3, col_kpi4, col_kpi5 = st.columns(5)
            with col_kpi1:
                st.metric("👥 Tổng khách", f"{tong_khach}")
            with col_kpi2:
                st.metric("📦 Cần soạn", f"{tong_can_soan:,}".replace(",", "."))
            with col_kpi3:
                st.metric("✅ Đã hoàn thành", f"{tong_da_soan:,}".replace(",", "."))
            with col_kpi4:
                st.metric("⚠️ Còn thiếu", f"{con_thieu:,}".replace(",", "."))
            with col_kpi5:
                st.metric("🚀 Tiến độ", f"{tien_do:.1f}%")
                
            st.progress(min(tien_do / 100.0, 1.0))

            st.markdown("---")
            
            st.write("### 📈 Tiến độ soạn đồ theo từng nhóm")
            summary_nhom = (
                df_latest.groupby("Nhóm")
                .agg(
                    Cần_soạn=("Cần soạn", "sum"),
                    Đã_soạn=("Đã soạn", "sum")
                )
                .reset_index()
            )
            summary_nhom["Còn thiếu"] = summary_nhom["Cần_soạn"] - summary_nhom["Đã_soạn"]
            summary_nhom["Còn thiếu"] = summary_nhom["Còn thiếu"].apply(lambda x: x if x > 0 else 0)
            
            col_chart, col_table = st.columns([1.5, 1])
            with col_chart:
                st.write("**Biểu đồ lượng đồ đạc: Đã soạn vs Cần soạn**")
                st.bar_chart(summary_nhom.set_index("Nhóm")[["Cần_soạn", "Đã_soạn"]])
            with col_table:
                st.write("**Bảng số liệu chi tiết theo nhóm:**")
                st.dataframe(
                    summary_nhom,
                    use_container_width=True,
                    hide_index=True
                )
        else:
            st.info("Chưa có dữ liệu để tổng hợp thống kê. Hãy cập nhật tiến độ ở mục Soạn đồ.")
