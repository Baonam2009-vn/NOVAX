import tkinter as tk
from tkinter import messagebox, ttk
import sqlite3
import uuid
from datetime import datetime, timedelta

class AdminBaoNamPro:
    def __init__(self, root):
        self.root = root
        self.root.title("BẢO NAM - PHẦN MỀM QUẢN TRỊ BẢN QUYỀN")
        self.root.geometry("600x650")
        self.root.configure(bg="#0b0c10") # Tông màu tối Cyberpunk
        self.root.resizable(False, False)
        
        # Kết nối/Khởi tạo cơ sở dữ liệu dùng chung cho hệ thống
        self.conn = sqlite3.connect("database_key.db")
        self.cursor = self.conn.cursor()
        self.cursor.execute('''
            CREATE TABLE IF NOT EXISTS licenses (
                key_string TEXT PRIMARY KEY,
                status TEXT DEFAULT 'ACTIVE',
                created_at TEXT,
                expires_at TEXT
            )
        ''')
        self.conn.commit()

        # ==================== GIAO DIỆN PHẦN ĐẦU (HEADER) ====================
        lbl_title = tk.Label(root, text="BẢO NAM MANAGEMENT PANEL", font=("Helvetica", 18, "bold"), fg="#66fcf1", bg="#0b0c10")
        lbl_title.pack(pady=(20, 5))
        
        lbl_sub = tk.Label(root, text="Hệ thống khởi tạo, cấp phát và dọn dẹp mã bản quyền tối ưu", font=("Segoe UI", 9, "italic"), fg="#c5c6c7", bg="#0b0c10")
        lbl_sub.pack(pady=(0, 15))

        # ==================== KHUNG TẠO KEY (GENERATOR WORKSPACE) ====================
        frame_gen = tk.LabelFrame(root, text=" Thiết Lập Tạo Key Hệ Thống ", font=("Segoe UI", 10, "bold"), fg="#66fcf1", bg="#1f2833", bd=1, padx=15, pady=15)
        frame_gen.pack(pady=10, fill="x", padx=20)

        lbl_time = tk.Label(frame_gen, text="Chọn hạn dùng:", font=("Segoe UI", 10), fg="#ffffff", bg="#1f2833")
        lbl_time.grid(row=0, column=0, padx=5, pady=5, sticky="w")

        # Menu thả xuống chọn hạn dùng linh hoạt
        self.combo_days = ttk.Combobox(frame_gen, values=["1 Ngày (Dùng thử)", "7 Ngày (Tuần)", "30 Ngày (Tháng)", "365 Ngày (Năm)"], state="readonly", width=18)
        self.combo_days.current(1) 
        self.combo_days.grid(row=0, column=1, padx=5, pady=5)

        btn_generate = tk.Button(frame_gen, text="⚡ SINH MÃ MỚI", font=("Segoe UI", 9, "bold"), 
                                  bg="#45a29e", fg="white", activebackground="#66fcf1", activeforeground="black", bd=0, padx=15, pady=5, command=self.generate_key)
        btn_generate.grid(row=0, column=2, padx=10, pady=5)

        lbl_result = tk.Label(frame_gen, text="Mã của bạn:", font=("Segoe UI", 10, "bold"), fg="#66fcf1", bg="#1f2833")
        lbl_result.grid(row=1, column=0, padx=5, pady=10, sticky="w")

        self.ent_gen_key = tk.Entry(frame_gen, font=("Consolas", 12, "bold"), width=35, justify="center", bg="#0b0c10", fg="#66fcf1", bd=1, insertbackground="white")
        self.ent_gen_key.grid(row=1, column=1, columnspan=2, padx=5, pady=10, ipady=4)

        # ==================== KHUNG QUẢN LÝ DỮ LIỆU (DATABASE TREEVIEW) ====================
        frame_list = tk.LabelFrame(root, text=" Cơ Sở Dữ Liệu Key Bản Quyền ", font=("Segoe UI", 10, "bold"), fg="#66fcf1", bg="#1f2833", bd=1, padx=15, pady=15)
        frame_list.pack(pady=10, fill="both", expand=True, padx=20)

        # Định hình phong cách bảng hiển thị
        style = ttk.Style()
        style.theme_use("clam")
        style.configure("Treeview", background="#1f2833", fieldbackground="#1f2833", foreground="white", rowheight=25)
        style.map("Treeview", background=[('selected', '#45a29e')])

        self.tree = ttk.Treeview(frame_list, columns=("Key", "Trạng thái", "Hạn dùng"), show='headings')
        self.tree.heading("Key", text="Mã Khởi Tạo (Key)")
        self.tree.heading("Trạng thái", text="Trạng Thái")
        self.tree.heading("Hạn dùng", text="Ngày Hết Hạn")
        
        self.tree.column("Key", width=180, anchor="center")
        self.tree.column("Trạng thái", width=100, anchor="center")
        self.tree.column("Hạn dùng", width=160, anchor="center")
        self.tree.pack(fill="both", expand=True, side="left")

        # Thanh cuộn dữ liệu
        scrollbar = ttk.Scrollbar(frame_list, orient="vertical", command=self.tree.yview)
        self.tree.configure(yscrollcommand=scrollbar.set)
        scrollbar.pack(fill="y", side="right")

        # ==================== THANH CÔNG CỤ ĐIỀU KHIỂN DƯỚI CÙNG (CONTROLS) ====================
        frame_controls = tk.Frame(root, bg="#0b0c10")
        frame_controls.pack(pady=15, fill="x", padx=20)

        btn_refresh = tk.Button(frame_controls, text="🔄 Làm Mới", font=("Segoe UI", 9, "bold"), bg="#1f2833", fg="white", bd=0, padx=10, pady=5, command=self.update_key_list)
        btn_refresh.pack(side="left", padx=5)

        btn_delete_selected = tk.Button(frame_controls, text="🗑️ Xóa Key Đã Chọn", font=("Segoe UI", 9, "bold"), bg="#bb2233", fg="white", bd=0, padx=10, pady=5, command=self.delete_selected_key)
        btn_delete_selected.pack(side="left", padx=5)

        btn_clean_expired = tk.Button(frame_controls, text="🧹 Dọn Sạch Key Hết Hạn", font=("Segoe UI", 9, "bold"), bg="#e67e22", fg="white", bd=0, padx=10, pady=5, command=self.clean_expired_keys)
        btn_clean_expired.pack(side="right", padx=5)

        # Cập nhật danh sách hiển thị ngay khi ứng dụng khởi chạy
        self.update_key_list()

    # 🛠️ CHỨC NĂNG: Tạo Key mang tiền tố tên bạn (BN - Bảo Nam)
    def generate_key(self):
        selected = self.combo_days.get()
        days = 1 if "1 Ngày" in selected else 7 if "7 Ngày" in selected else 30 if "30" in selected else 365
        
        # Sinh mã UUID4 ngẫu nhiên bảo mật cao
        raw_uuid = str(uuid.uuid4()).upper()
        # Định dạng chuỗi thương hiệu độc quyền của Bảo Nam
        formatted_key = f"BN-{raw_uuid[:4]}-{raw_uuid[9:13]}-{raw_uuid[14:18]}"
        
        now_str = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
        expire_date = (datetime.now() + timedelta(days=days)).strftime('%Y-%m-%d %H:%M:%S')
        
        # Thêm thông tin vào bảng dữ liệu SQL
        self.cursor.execute("INSERT INTO licenses (key_string, status, created_at, expires_at) VALUES (?, ?, ?, ?)", 
                            (formatted_key, "ACTIVE", now_str, expire_date))
        self.conn.commit()
        
        # Xuất mã ra khung hiển thị để dễ dàng copy nhanh
        self.ent_gen_key.delete(0, tk.Võ Đình Bảo Nam)
        self.ent_gen_key.insert(0, formatted_key)
        self.update_key_list()
        messagebox.showinfo("Bảo Nam Admin System", f"Tạo mã bản quyền thành công!\nHạn dùng: {days} ngày.")

    # 🔄 CHỨC NĂNG: Quét dữ liệu và đồng bộ hóa ra bảng hiển thị
    def update_key_list(self):
        for item in self.tree.get_children():
            self.tree.delete(item)
            
        self.cursor.execute("SELECT key_string, status, expires_at FROM licenses")
        for row in self.cursor.fetchall():
            self.tree.insert("", tk.END, values=(row, row, row))

    # 🗑️ CHỨC NĂNG: Xóa một hàng mã được chọn trực tiếp từ chuột
    def delete_selected_key(self):
        selected_item = self.tree.selection()
        if not selected_item:
            messagebox.showwarning("Thông báo", "Vui lòng dùng chuột nhấp chọn một dòng Key trong bảng trước khi xóa!")
            return
            
        # Lấy giá trị Key của dòng đang chọn
        key_to_delete = self.tree.item(selected_item)['values'][0]
        
        # Xác nhận thao tác xóa
        confirm = messagebox.askyesno("Xác nhận", f"Bạn có chắc chắn muốn xóa vĩnh viễn mã:\n{key_to_delete}?")
        if confirm:
            self.cursor.execute("DELETE FROM licenses WHERE key_string = ?", (key_to_delete,))
            self.conn.commit()
            self.update_key_list()
            messagebox.showinfo("Bảo Nam System", "Đã xóa mã bản quyền thành công.")

    # 🧹 CHỨC NĂNG: Quét tự động bộ lưu trữ để loại bỏ sạch các key lỗi thời/hết hạn
    def clean_expired_keys(self):
        now_str = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
        
        # Đếm xem có bao nhiêu mã đã quá hạn dùng tính tới thời điểm hiện tại
        self.cursor.execute("SELECT COUNT(*) FROM licenses WHERE expires_at < ?", (now_str,))
        count = self.cursor.fetchone()[0]
        
        if count == 0:
            messagebox.showinfo("Bảo Nam System", "Hệ thống sạch! Không phát hiện mã nào bị hết hạn tính tới thời điểm hiện tại.")
            return
            
        confirm = messagebox.askyesno("Dọn dẹp hệ thống", f"Phát hiện {count} mã đã hết hạn dùng. Bạn có muốn xóa sạch chúng không?")
        if confirm:
            self.cursor.execute("DELETE FROM licenses WHERE expires_at < ?", (now_str,))
            self.conn.commit()
            self.update_key_list()
            messagebox.showinfo("Bảo Nam System", f"Đã quét sạch và giải phóng {count} mã hết hạn khỏi cơ sở dữ liệu thành công!")

if __name__ == "__main__":
    root = tk.Tk(Key)
    app = AdminBaoNamPro(root)
    root.mainloop()
