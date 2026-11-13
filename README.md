"""
Hotel Management System - Tkinter GUI
Group Members: Adarsh Sharma, Anuvanshika, Chaitanya, Janani
College Project - Python with SQLite + Tkinter
"""

import sqlite3
import datetime
import tkinter as tk
from tkinter import ttk, messagebox, font


# ─────────────────────────────────────────
# THEME CONSTANTS
# ─────────────────────────────────────────

BG_DARK      = "#0F1923"   # deep navy
BG_CARD      = "#1A2535"   # card background
BG_SIDEBAR   = "#111D2B"   # sidebar
ACCENT       = "#C9A84C"   # gold accent
ACCENT_DARK  = "#A07830"
TEXT_MAIN    = "#EAE6DC"   # warm white
TEXT_MUTED   = "#8A96A8"
TEXT_DARK    = "#0F1923"
SUCCESS      = "#4CAF82"
DANGER       = "#E05C5C"
INFO         = "#4A90C4"
BORDER       = "#263347"

FONT_TITLE  = ("Georgia", 22, "bold")
FONT_HEADER = ("Georgia", 13, "bold")
FONT_LABEL  = ("Helvetica", 10)
FONT_BOLD   = ("Helvetica", 10, "bold")
FONT_SMALL  = ("Helvetica", 9)
FONT_MONO   = ("Courier", 10)


# ─────────────────────────────────────────
# DATABASE
# ─────────────────────────────────────────

def setup_db():
    conn = sqlite3.connect("hotel.db")
    cur = conn.cursor()

    # 1. Core Operational Tables
    cur.execute("""CREATE TABLE IF NOT EXISTS rooms (
        room_no INTEGER PRIMARY KEY, 
        room_type TEXT NOT NULL,
        price REAL NOT NULL, 
        status TEXT DEFAULT 'Available',
        hk_status TEXT DEFAULT 'Clean')""") 

    cur.execute("""CREATE TABLE IF NOT EXISTS customers (
        customer_id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL, phone TEXT NOT NULL,
        country TEXT DEFAULT 'India', meal_plan TEXT DEFAULT 'BB')""")

    cur.execute("""CREATE TABLE IF NOT EXISTS bookings (
        booking_id INTEGER PRIMARY KEY AUTOINCREMENT,
        customer_id INTEGER, room_no INTEGER,
        check_in TEXT, check_out TEXT,
        adults INTEGER DEFAULT 1, children INTEGER DEFAULT 0,
        status TEXT DEFAULT 'Booked',
        FOREIGN KEY(customer_id) REFERENCES customers(customer_id),
        FOREIGN KEY(room_no) REFERENCES rooms(room_no))""")

    # 2. Billing Table (with Revenue Tracking)
    cur.execute("""CREATE TABLE IF NOT EXISTS bills (
        bill_id INTEGER PRIMARY KEY AUTOINCREMENT,
        booking_id INTEGER, 
        total_amt REAL, 
        paid TEXT DEFAULT 'No',
        payment_date DATE, -- This is crucial for monthly revenue reports
        FOREIGN KEY(booking_id) REFERENCES bookings(booking_id))""")

    # Migration: Add payment_date if it's an old database that doesn't have it
    try:
        cur.execute("ALTER TABLE bills ADD COLUMN payment_date DATE")
    except sqlite3.OperationalError:
        pass # Column already exists

    # 3. Financial Tracking (Expenses/Loss)
    cur.execute("""CREATE TABLE IF NOT EXISTS expenses (
        expense_id INTEGER PRIMARY KEY AUTOINCREMENT,
        category TEXT,       -- e.g., 'Laundry', 'Salaries', 'Electricity'
        amount REAL,
        expense_date DATE DEFAULT CURRENT_DATE)""")

    # 4. Initial Room Setup
    cur.execute("SELECT COUNT(*) FROM rooms")
    if cur.fetchone()[0] == 0:
        rooms_data = [
            (101,'Single',350.0), (102,'Single',350.0), (103,'Double',700.0),
            (104,'Double',700.0), (105,'Suite',1000.0), (201,'Single',350.0),
            (202,'Double',700.0), (203,'Suite',1000.0)
        ]
        cur.executemany("INSERT INTO rooms (room_no, room_type, price) VALUES (?,?,?)", rooms_data)
        print("Database initialized with default rooms.")

    conn.commit()
    conn.close()

def calculate_stay_price(room_type, nights):
     # Rule: Single (350), Double (700), Suite (1000)
    # Weekly: Single (2000), Double (3500)
    # Monthly (30+ days): Single (8000), Double (10000)
    # Convert nights to integer just in case a string was passed
    try:
        nights = int(nights)
    except:
        return 0
        
    total = 0
    rem = nights

    if room_type == 'Single':
        months, rem = divmod(rem, 30)
        weeks, days = divmod(rem, 7)
        total = (months * 8000) + (weeks * 2000) + (days * 350)
        
    elif room_type == 'Double':
        months, rem = divmod(rem, 30)
        weeks, days = divmod(rem, 7)
        total = (months * 10000) + (weeks * 3500) + (days * 700)
        
    elif room_type == 'Suite':
        months, rem = divmod(rem, 30)
        weeks, days = divmod(rem, 7)
        total = (months * 25000) + (weeks * 6000) + (days * 1000)
        
    else:
        # Fallback for Deluxe or other types
        total = nights * 1000
        
    return total

 

def get_conn():
    return sqlite3.connect("hotel.db")


# ─────────────────────────────────────────
# HELPER WIDGETS
# ─────────────────────────────────────────

def styled_frame(parent, bg=BG_CARD, padx=0, pady=0):
    return tk.Frame(parent, bg=bg, padx=padx, pady=pady)

def label(parent, text, font=FONT_LABEL, fg=TEXT_MAIN, bg=BG_CARD, **kw):
    return tk.Label(parent, text=text, font=font, fg=fg, bg=bg, **kw)

def entry(parent, width=24, textvariable=None):
    e = tk.Entry(parent, width=width, font=FONT_LABEL,
                 bg="#1F2E40", fg=TEXT_MAIN, insertbackground=ACCENT,
                 relief="flat", bd=0, highlightthickness=1,
                 highlightbackground=BORDER, highlightcolor=ACCENT)
    if textvariable:
        e.config(textvariable=textvariable)
    return e

def combo(parent, values, textvariable=None, width=22):
    style = ttk.Style()
    style.theme_use("clam")
    style.configure("Hotel.TCombobox",
        fieldbackground="#1F2E40", background="#1F2E40",
        foreground=TEXT_MAIN, arrowcolor=ACCENT,
        bordercolor=BORDER, lightcolor=BORDER, darkcolor=BORDER)
    
    # Remove state="readonly" to allow typing
    c = ttk.Combobox(parent, values=values, width=width,
                     font=FONT_LABEL, style="Hotel.TCombobox") 
    if textvariable:
        c.config(textvariable=textvariable)
    return c

def gold_btn(parent, text, command, width=16, bg=ACCENT, fg=TEXT_DARK):
    b = tk.Button(parent, text=text, command=command,
                  font=FONT_BOLD, bg=bg, fg=fg,
                  activebackground=ACCENT_DARK, activeforeground=TEXT_DARK,
                  relief="flat", cursor="hand2", padx=10, pady=6, width=width)
    b.bind("<Enter>", lambda e: b.config(bg=ACCENT_DARK))
    b.bind("<Leave>", lambda e: b.config(bg=bg))
    return b

def danger_btn(parent, text, command, width=14):
    return gold_btn(parent, text, command, width=width, bg=DANGER, fg=TEXT_MAIN)

def section_title(parent, text):
    f = tk.Frame(parent, bg=BG_CARD)
    tk.Label(f, text=text, font=FONT_HEADER, fg=ACCENT, bg=BG_CARD).pack(side="left")
    tk.Frame(f, bg=ACCENT, height=2).pack(side="left", fill="x", expand=True, padx=(10,0), pady=8)
    return f

def build_treeview(parent, columns, col_widths=None):
    style = ttk.Style()
    style.configure("Hotel.Treeview",
        background=BG_DARK, foreground=TEXT_MAIN,
        fieldbackground=BG_DARK, rowheight=26,
        font=FONT_SMALL, borderwidth=0)
    style.configure("Hotel.Treeview.Heading",
        background=BG_CARD, foreground=ACCENT,
        font=FONT_BOLD, relief="flat")
    style.map("Hotel.Treeview",
        background=[("selected", ACCENT)],
        foreground=[("selected", TEXT_DARK)])

    frame = tk.Frame(parent, bg=BG_DARK)
    scroll = ttk.Scrollbar(frame, orient="vertical")
    tv = ttk.Treeview(frame, columns=columns, show="headings",
                      yscrollcommand=scroll.set, style="Hotel.Treeview")
    scroll.config(command=tv.yview)

    for i, col in enumerate(columns):
        w = col_widths[i] if col_widths else 120
        tv.heading(col, text=col)
        tv.column(col, width=w, anchor="center")

    tv.pack(side="left", fill="both", expand=True)
    scroll.pack(side="right", fill="y")
    return frame, tv

def form_row(parent, lbl_text, widget):
    row = tk.Frame(parent, bg=BG_CARD)
    tk.Label(row, text=lbl_text, font=FONT_LABEL, fg=TEXT_MUTED, bg=BG_CARD,
             width=18, anchor="w").pack(side="left", padx=(0,8))
    widget.pack(side="left")
    return row

def status_badge(parent, text):
    colors = {
        "Available": SUCCESS, "Occupied": DANGER,
        "Booked": INFO, "Checked-Out": TEXT_MUTED,
        "Cancelled": DANGER, "Yes": SUCCESS, "No": DANGER, "Pending": ACCENT
    }
    color = colors.get(text, TEXT_MUTED)
    return tk.Label(parent, text=f"  {text}  ", font=FONT_SMALL,
                    bg=color, fg=TEXT_DARK if text in ("Available","Yes") else TEXT_MAIN,
                    relief="flat", padx=4)


# ─────────────────────────────────────────
# MAIN APPLICATION
# ─────────────────────────────────────────

class HotelApp(tk.Tk):
    def __init__(self):
        super().__init__()
        self.title("Hotel Management System")
        self.geometry("1100x700")
        self.minsize(900, 600)
        self.configure(bg=BG_DARK)
        self.resizable(True, True)
        setup_db()
        self._build_layout()
        self._show_page("dashboard")

    def _build_layout(self):
        # ── Sidebar ──
        self.sidebar = tk.Frame(self, bg=BG_SIDEBAR, width=200)
        self.sidebar.pack(side="left", fill="y")
        self.sidebar.pack_propagate(False)

        # Logo area
        logo_f = tk.Frame(self.sidebar, bg=BG_SIDEBAR, pady=24)
        logo_f.pack(fill="x")
        tk.Label(logo_f, text="🏨", font=("Helvetica", 28), bg=BG_SIDEBAR, fg=ACCENT).pack()
        tk.Label(logo_f, text="GRAND HOTEL", font=("Georgia", 11, "bold"),
                 bg=BG_SIDEBAR, fg=ACCENT).pack()
        tk.Label(logo_f, text="Management Suite", font=("Helvetica", 8),
                 bg=BG_SIDEBAR, fg=TEXT_MUTED).pack()
        tk.Frame(self.sidebar, bg=BORDER, height=1).pack(fill="x", padx=16)

        # Nav buttons
        nav_items = [
            ("dashboard", "⊞  Dashboard"),
            ("rooms",     "🚪  Rooms"),
            ("customers", "👤  Customers"),
            ("bookings",  "📋  Bookings"),
            ("billing",   "💳  Billing"),
            ("housekeeping", "🧹  Housekeeping"),
            ("analytics", "📈   Finance & Analytics"),
        ]
        self.nav_btns = {}
        nav_frame = tk.Frame(self.sidebar, bg=BG_SIDEBAR, pady=16)
        nav_frame.pack(fill="x")
        for key, text in nav_items:
            btn = tk.Button(nav_frame, text=text, font=FONT_LABEL,
                            bg=BG_SIDEBAR, fg=TEXT_MUTED, relief="flat",
                            anchor="w", padx=20, pady=10, cursor="hand2",
                            activebackground=BG_CARD, activeforeground=ACCENT,
                            command=lambda k=key: self._show_page(k))
            btn.pack(fill="x", pady=1)
            self.nav_btns[key] = btn

        # ── Content area ──
        self.content = tk.Frame(self, bg=BG_DARK)
        self.content.pack(side="left", fill="both", expand=True)

        self.pages = {}
        for PageClass, key in [
            (DashboardPage, "dashboard"),
            (RoomsPage,     "rooms"),
            (CustomersPage, "customers"),
            (BookingsPage,  "bookings"),
            (BillingPage,   "billing"),
            (HousekeepingPage, "housekeeping"),
            (AnalyticsPage,  "analytics"),
        ]:
            page = PageClass(self.content, self)
            page.place(relx=0, rely=0, relwidth=1, relheight=1)
            self.pages[key] = page
           

    def refresh(self):
        self.tv.delete(*self.tv.get_children())
        conn = get_conn(); cur = conn.cursor()
        cur.execute("SELECT room_no, room_type, status, hk_status FROM rooms")
        for row in cur.fetchall():
            tag = "dirty" if row[3] == "Dirty" else "clean"
            if row[3] == "Service Requested": tag = "request"
            self.tv.insert("", "end", values=row, tags=(tag,))
        
        self.tv.tag_configure("dirty", background="#2E1A1A") # Reddish
        self.tv.tag_configure("clean", background="#152218") # Greenish
        self.tv.tag_configure("request", background="#2E2A12") # Yellowish
        conn.close()

    def _update_hk(self, new_status):
        sel = self.tv.selection()
        if not sel: return
        room_no = self.tv.item(sel[0])['values'][0]
        conn = get_conn(); cur = conn.cursor()
        cur.execute("UPDATE rooms SET hk_status=? WHERE room_no=?", (new_status, room_no))
        conn.commit(); conn.close()
        self.refresh()

    def _show_page(self, key):
        for k, btn in self.nav_btns.items():
            if k == key:
                btn.config(bg=BG_CARD, fg=ACCENT, font=FONT_BOLD)
            else:
                btn.config(bg=BG_SIDEBAR, fg=TEXT_MUTED, font=FONT_LABEL)
        self.pages[key].tkraise()
        if hasattr(self.pages[key], "refresh"):
            self.pages[key].refresh()
    
    def get_monthly_report(self):
   
    #"""Calculates Revenue vs Expenses for the current month"""
   
      conn = get_conn(); 
      cur = conn.cursor()
    
      # Current month/year in YYYY-MM format
      current_month = datetime.date.today().strftime('%Y-%m')
    
    # 1. Total Revenue (Profit from paid bills)
      cur.execute("SELECT SUM(total_amt) FROM bills WHERE paid='Yes' AND strftime('%Y-%m', payment_date) = ?", (current_month,))
      revenue = cur.fetchone()[0] or 0.0
    
    # 2. Total Expenses (Loss/Costs)
      cur.execute("SELECT SUM(amount) FROM expenses WHERE strftime('%Y-%m', expense_date) = ?", (current_month,))
      loss = cur.fetchone()[0] or 0.0
    
      net_profit = revenue - loss
      conn.close()
    
      return revenue, loss, net_profit
    
class AnalyticsPage(tk.Frame):
    def __init__(self, parent, app):
        super().__init__(parent, bg=BG_DARK)
        self.app = app
        self._build_ui()

    def _build_ui(self):
        # Header
        hdr = tk.Frame(self, bg=BG_DARK, pady=20, padx=30)
        hdr.pack(fill="x")
        tk.Label(hdr, text="Monthly Performance & Backups", font=FONT_TITLE, fg=ACCENT, bg=BG_DARK).pack(side="left")
        
        # Action Bar (Add Expense Button)
        bar = tk.Frame(self, bg=BG_DARK, padx=30)
        bar.pack(fill="x")
        gold_btn(bar, "💸 Record Expense", self.add_expense_dialog).pack(side="left")

        # Stats Cards (Top Row)
        self.stats_frame = tk.Frame(self, bg=BG_DARK, padx=30)
        self.stats_frame.pack(fill="x", pady=20)

        # Main Table (Historical Comparison)
        body = tk.Frame(self, bg=BG_DARK, padx=30)
        body.pack(fill="both", expand=True)
        cols = ["Month", "Revenue", "Expenses (Loss)", "Net Profit", "Growth %"]
        f, self.tv = build_treeview(body, cols, [120, 150, 150, 150, 120])
        f.pack(fill="both", expand=True)

    def refresh(self):
        """Called when the user clicks the Analytics button in the sidebar"""
        self.tv.delete(*self.tv.get_children())
        
        # 1. Update Top Stats Cards
        for widget in self.stats_frame.winfo_children(): widget.destroy()
        rev, loss, profit = self.app.get_monthly_report()
        
        self._create_card("Total Revenue", f"₹{rev:,.2f}", SUCCESS)
        self._create_card("Total Expenses", f"₹{loss:,.2f}", "#FF5555")
        self._create_card("Net Profit", f"₹{profit:,.2f}", ACCENT)

        # 2. Populate Historical Comparison Table
        self._load_history()

    def _create_card(self, label, value, color):
        card = tk.Frame(self.stats_frame, bg=BG_CARD, padx=20, pady=15, highlightthickness=1, highlightbackground=BORDER)
        card.pack(side="left", padx=10, expand=True, fill="x")
        tk.Label(card, text=label, font=FONT_SMALL, fg=TEXT_MUTED, bg=BG_CARD).pack()
        tk.Label(card, text=value, font=FONT_BOLD, fg=color, bg=BG_CARD).pack()

    def _load_history(self):
        conn = get_conn(); cur = conn.cursor()
        prev_profit = 0
        
        # Look back at the last 6 months
        for i in range(5, -1, -1):
            # Calculate Year-Month (e.g., 2026-04)
            d = datetime.date.today() - datetime.timedelta(days=i*30)
            m_str = d.strftime('%Y-%m')
            
            # Get Revenue
            cur.execute("SELECT SUM(total_amt) FROM bills WHERE paid='Yes' AND strftime('%Y-%m', payment_date) = ?", (m_str,))
            r = cur.fetchone()[0] or 0.0
            
            # Get Loss
            cur.execute("SELECT SUM(amount) FROM expenses WHERE strftime('%Y-%m', expense_date) = ?", (m_str,))
            l = cur.fetchone()[0] or 0.0
            
            p = r - l
            growth = "—"
            if prev_profit != 0:
                perc = ((p - prev_profit) / abs(prev_profit)) * 100
                growth = f"{perc:+.1f}%"
            
            self.tv.insert("", "end", values=(m_str, f"₹{r:,.2f}", f"₹{l:,.2f}", f"₹{p:,.2f}", growth))
            prev_profit = p
        conn.close()

    def add_expense_dialog(self):
        win = tk.Toplevel(self)
        win.title("Record Hotel Expense")
        win.geometry("400x350")
        win.configure(bg=BG_CARD)
        win.grab_set() # Focus on this window

        v_cat = tk.StringVar()
        v_amt = tk.StringVar()

        tk.Label(win, text="Category", font=FONT_SMALL, bg=BG_CARD, fg=TEXT_MUTED).pack(pady=(20,0))
        entry(win, textvariable=v_cat).pack(pady=5)
        tk.Label(win, text="e.g., Laundry, Electricity, Salaries", font=("Helvetica", 8), bg=BG_CARD, fg=TEXT_MUTED).pack()

        tk.Label(win, text="Amount (₹)", font=FONT_SMALL, bg=BG_CARD, fg=TEXT_MUTED).pack(pady=(20,0))
        entry(win, textvariable=v_amt).pack(pady=5)

        def save():
            try:
                category = v_cat.get().strip()
                amount = float(v_amt.get())
                
                if not category or amount <= 0:
                    raise ValueError

                conn = get_conn(); cur = conn.cursor()
                # payment_date is handled by DEFAULT CURRENT_DATE in DB setup
                cur.execute("INSERT INTO expenses (category, amount) VALUES (?,?)", (category, amount))
                conn.commit(); conn.close()
                
                messagebox.showinfo("Saved", "Expense recorded for monthly analysis.")
                win.destroy()
                # If AnalyticsPage is open, it will need a refresh
                if hasattr(self.app, "pages") and "analytics" in self.app.pages:
                    self.app.pages["analytics"].refresh()
            except ValueError:
                messagebox.showerror("Error", "Please enter a valid category and positive amount.")

        gold_btn(win, "💾 Save Expense", save, width=20).pack(pady=30)


# ─────────────────────────────────────────
# DASHBOARD PAGE
# ─────────────────────────────────────────

class DashboardPage(tk.Frame):
    def __init__(self, parent, app):
        super().__init__(parent, bg=BG_DARK)
        self.app = app
        self._build()

    def _build(self):
        # Header
        hdr = tk.Frame(self, bg=BG_DARK, pady=20, padx=30)
        hdr.pack(fill="x")
        tk.Label(hdr, text="Dashboard", font=FONT_TITLE, fg=ACCENT, bg=BG_DARK).pack(side="left")
        self.date_lbl = tk.Label(hdr, text="", font=FONT_SMALL, fg=TEXT_MUTED, bg=BG_DARK)
        self.date_lbl.pack(side="right", pady=8)
        self._update_clock()

        # Stat cards
        self.cards_frame = tk.Frame(self, bg=BG_DARK, padx=30)
        self.cards_frame.pack(fill="x", pady=(0, 20))

        # Recent bookings
        body = tk.Frame(self, bg=BG_DARK, padx=30)
        body.pack(fill="both", expand=True)
        section_title(body, "Recent Bookings").pack(fill="x", pady=(0,10))
        cols = ["Booking ID", "Guest", "Room", "Check-In", "Check-Out", "Status"]
        widths = [90, 160, 70, 110, 110, 110]
        tv_frame, self.tv = build_treeview(body, cols, widths)
        tv_frame.pack(fill="both", expand=True, pady=(0,20))

    def _update_clock(self):
        now = datetime.datetime.now()
        self.date_lbl.config(text=now.strftime("  %A, %d %B %Y   %H:%M"))
        self.after(1000, self._update_clock)

    def _stat_card(self, parent, icon, title, value, color):
        f = tk.Frame(parent, bg=BG_CARD, padx=20, pady=16,
                     highlightthickness=1, highlightbackground=BORDER)
        f.pack(side="left", padx=(0,14), pady=4, fill="y")
        tk.Label(f, text=icon, font=("Helvetica", 24), bg=BG_CARD, fg=color).pack(anchor="w")
        tk.Label(f, text=str(value), font=("Georgia", 22, "bold"),
                 bg=BG_CARD, fg=color).pack(anchor="w")
        tk.Label(f, text=title, font=FONT_SMALL, bg=BG_CARD, fg=TEXT_MUTED).pack(anchor="w")

    def refresh(self):
        # Clear old cards
        for w in self.cards_frame.winfo_children():
            w.destroy()

        conn = get_conn(); cur = conn.cursor()
        cur.execute("SELECT COUNT(*) FROM rooms WHERE status='Available'")
        avail = cur.fetchone()[0]
        cur.execute("SELECT COUNT(*) FROM rooms WHERE status='Occupied'")
        occ = cur.fetchone()[0]
        cur.execute("SELECT COUNT(*) FROM bookings WHERE status='Booked'")
        active = cur.fetchone()[0]
        cur.execute("SELECT COALESCE(SUM(total_amt),0) FROM bills WHERE paid='Yes'")
        revenue = cur.fetchone()[0]
        conn.close()

        self._stat_card(self.cards_frame, "🚪", "Available Rooms", avail, SUCCESS)
        self._stat_card(self.cards_frame, "🏠", "Occupied Rooms",  occ,   DANGER)
        self._stat_card(self.cards_frame, "📋", "Active Bookings", active, INFO)
        self._stat_card(self.cards_frame, "💰", "Total Revenue",
                        f"₹{revenue:,.0f}", ACCENT)

        # Recent bookings
        self.tv.delete(*self.tv.get_children())
        conn = get_conn(); cur = conn.cursor()
        cur.execute("""SELECT b.booking_id, c.name, b.room_no, b.check_in, b.check_out, b.status
            FROM bookings b JOIN customers c ON b.customer_id=c.customer_id
            ORDER BY b.booking_id DESC LIMIT 15""")
        for row in cur.fetchall():
            tag = {"Booked": "info", "Checked-Out": "muted", "Cancelled": "danger"}.get(row[5], "")
            self.tv.insert("", "end", values=row, tags=(tag,))
        self.tv.tag_configure("info",   background="#1A2E40")
        self.tv.tag_configure("muted",  background="#1A2535")
        self.tv.tag_configure("danger", background="#2E1A1A")
        conn.close()


# ─────────────────────────────────────────
# ROOMS PAGE
# ─────────────────────────────────────────

class RoomsPage(tk.Frame):
    def __init__(self, parent, app):
        super().__init__(parent, bg=BG_DARK)
        self.app = app
        self._build()

    def _build(self):
        # Header
        hdr = tk.Frame(self, bg=BG_DARK, pady=20, padx=30)
        hdr.pack(fill="x")
        tk.Label(hdr, text="Room Management", font=FONT_TITLE, fg=ACCENT, bg=BG_DARK).pack(side="left")

        # Body
        body = tk.Frame(self, bg=BG_DARK)
        body.pack(fill="both", expand=True, padx=30)

        # Left: Table
        left = tk.Frame(body, bg=BG_DARK)
        left.pack(side="left", fill="both", expand=True, padx=(0,20))

        section_title(left, "All Rooms").pack(fill="x", pady=(0,10))
        cols = ["Room No", "Type", "Price/Night", "Status", "HK Status"]
        widths = [80, 100, 120, 110, 110]
        tv_frame, self.tv = build_treeview(left, cols, widths)
        tv_frame.pack(fill="both", expand=True)

        # Right: Form
        right = tk.Frame(body, bg=BG_CARD, width=280, padx=20, pady=20,
                         highlightthickness=1, highlightbackground=BORDER)
        right.pack(side="right", fill="y")
        right.pack_propagate(False)

        # --- Section 1: Add Room ---
        tk.Label(right, text="Add New Room", font=FONT_HEADER, fg=ACCENT, bg=BG_CARD).pack(anchor="w", pady=(0,14))
        self.v_room_no = tk.StringVar()
        self.v_room_type = tk.StringVar(value="Single")
        self.v_price = tk.StringVar()

        tk.Label(right, text="Room Number", font=FONT_SMALL, fg=TEXT_MUTED, bg=BG_CARD).pack(anchor="w")
        entry(right, textvariable=self.v_room_no).pack(fill="x", pady=(0,10))
        
        tk.Label(right, text="Room Type", font=FONT_SMALL, fg=TEXT_MUTED, bg=BG_CARD).pack(anchor="w")
        combo(right, ["Single", "Double", "Suite", "Deluxe"], self.v_room_type).pack(fill="x", pady=(0,10))

        tk.Label(right, text="Daily Price", font=FONT_SMALL, fg=TEXT_MUTED, bg=BG_CARD).pack(anchor="w")
        entry(right, textvariable=self.v_price).pack(fill="x", pady=(0,10))

        gold_btn(right, "➕ Add Room", self._add_room, width=22).pack(fill="x")

        # Divider
        tk.Frame(right, bg=BORDER, height=1).pack(fill="x", pady=20)

        # --- Section 2: Update Price by Type (FIXED HERE) ---
        tk.Label(right, text="Update Price by Type", font=FONT_HEADER, fg=ACCENT, bg=BG_CARD).pack(anchor="w", pady=(0,10))
        
        self.v_upd_type = tk.StringVar(value="Single") 
        self.v_upd_price = tk.StringVar()
        
        tk.Label(right, text="Select Type", font=FONT_SMALL, fg=TEXT_MUTED, bg=BG_CARD).pack(anchor="w")
        # Change state to allow writing as requested
        self.cb_upd = combo(right, ["Single", "Double", "Suite", "Deluxe"], self.v_upd_type)
        self.cb_upd.pack(fill="x", pady=(0,10))
        
        tk.Label(right, text="New Price/Night", font=FONT_SMALL, fg=TEXT_MUTED, bg=BG_CARD).pack(anchor="w")
        entry(right, textvariable=self.v_upd_price).pack(fill="x", pady=(0,10))
        
        gold_btn(right, "✏️ Update All", self._update_price, width=22).pack(fill="x")

    def _update_price(self):
        try:
            rtype = self.v_upd_type.get()
            price_val = float(self.v_upd_price.get())
        except ValueError:
            messagebox.showerror("Input Error", "Enter valid Price.")
            return

        conn = get_conn(); cur = conn.cursor()
        cur.execute("UPDATE rooms SET price=? WHERE room_type=?", (price_val, rtype))
        count = cur.rowcount
        
        if count > 0:
            conn.commit()
            messagebox.showinfo("Success", f"All {rtype} rooms updated to ₹{price_val}.")
            # IMPORTANT: We use v_upd_price here, not the old v_upd_no
            self.v_upd_price.set("")
            self.refresh()
        else:
            messagebox.showerror("Error", "No rooms of this type found.")
        conn.close()

    def refresh(self):
        self.tv.delete(*self.tv.get_children())
        conn = get_conn(); cur = conn.cursor()
        cur.execute("SELECT room_no, room_type, price, status, hk_status FROM rooms ORDER BY room_no")
        for row in cur.fetchall():
            self.tv.insert("", "end", values=(row[0], row[1], f"₹{row[2]}", row[3], row[4]))
        conn.close()

    def _add_room(self):
        try:
            rno = int(self.v_room_no.get())
            pr = float(self.v_price.get())
            rt = self.v_room_type.get()
            conn = get_conn(); cur = conn.cursor()
            cur.execute("INSERT INTO rooms (room_no, room_type, price) VALUES (?,?,?)", (rno, rt, pr))
            conn.commit(); conn.close()
            messagebox.showinfo("Success", "Room Added")
            self.refresh()
        except:
            messagebox.showerror("Error", "Could not add room. Check if Number exists.")


# ─────────────────────────────────────────
# CUSTOMERS PAGE
# ─────────────────────────────────────────

class CustomersPage(tk.Frame):
    def __init__(self, parent, app):
        super().__init__(parent, bg=BG_DARK)
        self.app = app
        self._build()

    def _build(self):
        hdr = tk.Frame(self, bg=BG_DARK, pady=20, padx=30)
        hdr.pack(fill="x")
        tk.Label(hdr, text="Customer Management", font=FONT_TITLE, fg=ACCENT, bg=BG_DARK).pack(side="left")

        body = tk.Frame(self, bg=BG_DARK)
        body.pack(fill="both", expand=True, padx=30)

        # Left table
        left = tk.Frame(body, bg=BG_DARK)
        left.pack(side="left", fill="both", expand=True, padx=(0,20))

        section_title(left, "All Customers").pack(fill="x", pady=(0,10))
        cols = ["ID", "Name", "Phone", "Country", "Meal Plan"]
        widths = [50, 160, 120, 100, 90]
        tv_frame, self.tv = build_treeview(left, cols, widths)
        tv_frame.pack(fill="both", expand=True)

        gold_btn(left, "📖 View Booking History", self._view_history, width=24).pack(pady=10, anchor="w")

        # Right form
        right = tk.Frame(body, bg=BG_CARD, width=260, padx=20, pady=20,
                         highlightthickness=1, highlightbackground=BORDER)
        right.pack(side="right", fill="y")
        right.pack_propagate(False)

        tk.Label(right, text="Add New Customer", font=FONT_HEADER, fg=ACCENT, bg=BG_CARD).pack(anchor="w", pady=(0,14))

        self.v_name    = tk.StringVar()
        self.v_phone   = tk.StringVar()
        self.v_country = tk.StringVar(value="India")
        self.v_meal    = tk.StringVar(value="BB")

        for lbl, var in [("Full Name", self.v_name), ("Phone Number", self.v_phone), ("Country", self.v_country)]:
            tk.Label(right, text=lbl, font=FONT_SMALL, fg=TEXT_MUTED, bg=BG_CARD).pack(anchor="w", pady=(8,2))
            entry(right, textvariable=var).pack(anchor="w", fill="x")

        tk.Label(right, text="Meal Plan", font=FONT_SMALL, fg=TEXT_MUTED, bg=BG_CARD).pack(anchor="w", pady=(8,2))
        combo(right, ["BB - Bed & Breakfast", "HB - Half Board", "FB - Full Board"],
              self.v_meal, width=22).pack(anchor="w")

        gold_btn(right, "➕  Add Customer", self._add_customer, width=22).pack(pady=(16,0), fill="x")

    def refresh(self):
        self.tv.delete(*self.tv.get_children())
        conn = get_conn(); cur = conn.cursor()
        cur.execute("SELECT customer_id, name, phone, country, meal_plan FROM customers ORDER BY customer_id DESC")
        for row in cur.fetchall():
            self.tv.insert("", "end", values=row)
        conn.close()

    def _add_customer(self):
        name    = self.v_name.get().strip()
        phone   = self.v_phone.get().strip()
        country = self.v_country.get().strip() or "India"
        meal    = self.v_meal.get().split(" ")[0]  # take 'BB', 'HB', or 'FB'
        if not name or not phone:
            messagebox.showerror("Error", "Name and Phone are required.")
            return
        conn = get_conn(); cur = conn.cursor()
        cur.execute("INSERT INTO customers (name, phone, country, meal_plan) VALUES (?,?,?,?)",
                    (name, phone, country, meal))
        conn.commit()
        cid = cur.lastrowid
        conn.close()
        messagebox.showinfo("Success", f"Customer added! ID: {cid}")
        self.v_name.set(""); self.v_phone.set("")
        self.refresh()

    def _view_history(self):
        sel = self.tv.selection()
        if not sel:
            messagebox.showinfo("Select Customer", "Please select a customer from the list.")
            return
        cid = self.tv.item(sel[0])["values"][0]
        win = tk.Toplevel(self)
        win.title(f"Booking History — Customer #{cid}")
        win.geometry("700x400")
        win.configure(bg=BG_DARK)
        tk.Label(win, text=f"Booking History for Customer ID: {cid}",
                 font=FONT_HEADER, fg=ACCENT, bg=BG_DARK).pack(pady=16, padx=20, anchor="w")
        cols = ["Booking ID", "Room", "Type", "Check-In", "Check-Out", "Status"]
        widths = [90, 70, 90, 110, 110, 110]
        f, tv = build_treeview(win, cols, widths)
        f.pack(fill="both", expand=True, padx=20, pady=(0,20))
        conn = get_conn(); cur = conn.cursor()
        cur.execute("""SELECT b.booking_id, r.room_no, r.room_type, b.check_in, b.check_out, b.status
            FROM bookings b JOIN rooms r ON b.room_no=r.room_no
            WHERE b.customer_id=? ORDER BY b.booking_id DESC""", (cid,))
        for row in cur.fetchall():
            tv.insert("", "end", values=row)
        conn.close()


# ─────────────────────────────────────────
# BOOKINGS PAGE
# ─────────────────────────────────────────

class BookingsPage(tk.Frame):
    def __init__(self, parent, app):
        super().__init__(parent, bg=BG_DARK)
        self.app = app
        self._build_ui()

    def _build_ui(self):
        # Header Area
        hdr = tk.Frame(self, bg=BG_DARK, pady=20, padx=30)
        hdr.pack(fill="x")
        tk.Label(hdr, text="Bookings Management", font=FONT_TITLE, fg=ACCENT, bg=BG_DARK).pack(side="left")
        
        # Action Bar
        bar = tk.Frame(self, bg=BG_DARK, padx=30)
        bar.pack(fill="x")
        gold_btn(bar, "➕ New Booking", self._new_booking_dialog).pack(side="left", padx=5)
        gold_btn(bar, "✔ Checkout Guest", self._checkout_dialog).pack(side="left", padx=5)
        danger_btn(bar, "❌ Cancel Booking", self._cancel_booking).pack(side="left", padx=5)
        # Main Table
        body = tk.Frame(self, bg=BG_DARK, padx=30)
        body.pack(fill="both", expand=True, pady=20)
        cols = ["ID", "Guest Name", "Room", "Check-In", "Check-Out", "Status"]
        f, self.tv = build_treeview(body, cols, [50, 180, 80, 110, 110, 110])
        f.pack(fill="both", expand=True)

    def _new_booking_dialog(self):
        win = tk.Toplevel(self)
        win.title("Room Booking & Payment POS")
        win.geometry("550x900")
        win.configure(bg=BG_CARD)
        win.grab_set()

        # --- Variables ---
        v_phone = tk.StringVar(); 
        v_name = tk.StringVar(); 
        v_country = tk.StringVar(value="India")
        v_room = tk.StringVar(); 
        v_ci = tk.StringVar(value=datetime.date.today().isoformat())
        v_days = tk.StringVar(value="1");
        v_pay_method = tk.StringVar(value="Cash")
        v_card_no = tk.StringVar(); 
        v_card_exp = tk.StringVar();
        v_card_name = tk.StringVar()
       
        self.found_cust_id = None 
        self.current_calculated_co = ""

        # Main Scrollable-like Container
        form = tk.Frame(win, bg=BG_CARD, padx=24)
        form.pack(fill="both", expand=True)

        # --- INTERNAL LOGIC FUNCTIONS ---
        
        def lookup_guest():
            p = v_phone.get().strip()
            if not (p.isdigit() and len(p) == 10):
                messagebox.showwarning("Invalid Phone", "Please enter a 10-digit phone number.")
                return
            c = get_conn(); cur = c.cursor()
            cur.execute("SELECT customer_id, name, country FROM customers WHERE phone=?", (p,))
            res = cur.fetchone(); c.close()
            if res:
                self.found_cust_id, name, country = res
                v_name.set(name); v_country.set(country)
                messagebox.showinfo("Found", f"Welcome back, {name}!")
            else:
                self.found_cust_id = None
                messagebox.showinfo("New Guest", "No record found. Please enter details below.")

        def handle_payment_change(*args):
            method = v_pay_method.get()
            # Clear previous card fields to redraw
            for widget in card_frame.winfo_children(): 
                widget.destroy()
            
            if method == "Debit Card":
                # Layout for Debit Card Manual Entry
                tk.Label(card_frame, text="Card Number", font=FONT_SMALL, fg=TEXT_MUTED, bg=BG_CARD).pack(anchor="w", pady=(5,0))
                entry(card_frame, textvariable=v_card_no).pack(fill="x")
                
                # Adding Expiry Date Field
                tk.Label(card_frame, text="Expiry Date (MM/YY)", font=FONT_SMALL, fg=TEXT_MUTED, bg=BG_CARD).pack(anchor="w", pady=(8,0))
                entry(card_frame, textvariable=v_card_exp).pack(fill="x")
            
            elif method == "Credit Card":
                # AUTO-CAPTURE Logic
                guest_name = v_name.get().strip() or "Valued Guest"
                v_card_no.set("4111-XXXX-XXXX-1234")
                v_card_exp.set("05/29")
                v_card_name.set(guest_name)
                
                tk.Label(card_frame, text=f"✔ Credit Card Linked to {guest_name}", fg=SUCCESS, bg=BG_CARD, font=FONT_BOLD).pack(anchor="w", pady=5)
                tk.Label(card_frame, text=f"Card: {v_card_no.get()}", fg=TEXT_MAIN, bg=BG_CARD).pack(anchor="w")
                tk.Label(card_frame, text=f"Expires: {v_card_exp.get()}", fg=TEXT_MAIN, bg=BG_CARD).pack(anchor="w")
            
            else:
                # For 'Cash', hide the frame or keep it empty
                pass

        v_pay_method.trace_add("write", handle_payment_change)

        def do_smart_book():
            try:
                name, phone = v_name.get().strip(), v_phone.get().strip()
                if not name or len(phone) != 10 or not v_room.get():
                    messagebox.showerror("Error", "Please fill Name, 10-digit Phone, and Room.")
                    return
                
                # Database setup
                conn = get_conn(); cur = conn.cursor()
                
                # CID Logic
                cid = None
                if self.found_cust_id:
                    cid = self.found_cust_id
                    cur.execute("UPDATE customers SET name=?, country=? WHERE customer_id=?", (name, v_country.get(), cid))
                else:
                    try:
                        cur.execute("INSERT INTO customers (name, phone, country) VALUES (?,?,?)", (name, phone, v_country.get()))
                        cid = cur.lastrowid
                    except:
                        cur.execute("SELECT customer_id FROM customers WHERE phone=?", (phone,))
                        cid = cur.fetchone()[0]

                # Booking & Bill Logic
                r_parts = v_room.get().split(" — ")
                r_no = int(r_parts[0].split(" ")[1])
                r_type = r_parts[1]
                amt = calculate_stay_price(r_type, int(v_days.get()))

                cur.execute("INSERT INTO bookings (customer_id, room_no, check_in, check_out) VALUES (?,?,?,?)", 
                            (cid, r_no, v_ci.get(), self.current_calculated_co))
                bid = cur.lastrowid
                
                # Billing with Payment Info
                cur.execute("INSERT INTO bills (booking_id, total_amt, paid) VALUES (?,?, 'No')", (bid, amt))
                cur.execute("UPDATE rooms SET status='Occupied' WHERE room_no=?", (r_no,))
                
                conn.commit(); conn.close()
                messagebox.showinfo("Success", f"Booked via {v_pay_method.get()}\nTotal Bill: ₹{amt}")
                win.destroy(); self.refresh()
            except Exception as e: messagebox.showerror("System Error", str(e))

        # --- UI LAYOUT (STEP-BY-STEP) ---

        # Step 1: Guest Info
        tk.Label(form, text="1. Guest Information", font=FONT_BOLD, fg=ACCENT, bg=BG_CARD).pack(anchor="w", pady=(10,5))
        sf = tk.Frame(form, bg=BG_CARD); sf.pack(fill="x")
        entry(sf, textvariable=v_phone, width=20).pack(side="left")
        gold_btn(sf, "🔍 Search", lookup_guest, width=8).pack(side="left", padx=5)

        tk.Label(form, text="Guest Name *", font=FONT_SMALL, fg=TEXT_MUTED, bg=BG_CARD).pack(anchor="w", pady=(8,2))
        entry(form, textvariable=v_name).pack(fill="x")

        # Step 2: Room & Stay
        tk.Label(form, text="2. Stay Details (Meals Free)", font=FONT_BOLD, fg=ACCENT, bg=BG_CARD).pack(anchor="w", pady=(20,5))
        
        conn = get_conn(); cur = conn.cursor()
        cur.execute("SELECT room_no, room_type, price FROM rooms WHERE status='Available'")
        room_strs = [f"Room {r[0]} — {r[1]} — ₹{r[2]}/night" for r in cur.fetchall()]
        conn.close()

        tk.Label(form, text="Select Room", font=FONT_SMALL, fg=TEXT_MUTED, bg=BG_CARD).pack(anchor="w")
        cb_r = ttk.Combobox(form, values=room_strs, textvariable=v_room); cb_r.pack(fill="x")
        if room_strs: cb_r.current(0)

        tk.Label(form, text="Number of Days", font=FONT_SMALL, fg=TEXT_MUTED, bg=BG_CARD).pack(anchor="w", pady=(8,2))
        entry(form, textvariable=v_days).pack(fill="x")

        # Stay Preview Box
        preview_f = tk.Frame(form, bg=BG_DARK, padx=10, pady=10); preview_f.pack(fill="x", pady=15)
        lbl_co = tk.Label(preview_f, text="", fg=SUCCESS, bg=BG_DARK, font=FONT_SMALL)
        lbl_co.pack(anchor="w")
        lbl_bill = tk.Label(preview_f, text="Total: ₹0.00", fg=ACCENT, bg=BG_DARK, font=FONT_BOLD)
        lbl_bill.pack(anchor="w")

        def update_preview(*args):
            try:
                d = int(v_days.get() or 0)
                co = (datetime.date.fromisoformat(v_ci.get()) + datetime.timedelta(days=d)).isoformat()
                self.current_calculated_co = co
                lbl_co.config(text=f"Check-out: {co}")
                if v_room.get():
                    rt = v_room.get().split(" — ")[1]
                    lbl_bill.config(text=f"Total: ₹{calculate_stay_price(rt, d):,.2f}")
            except: pass

        v_days.trace_add("write", update_preview)
        v_room.trace_add("write", update_preview); update_preview()

        # Step 3: Payment
        tk.Label(form, text="3. Payment Method", font=FONT_BOLD, fg=ACCENT, bg=BG_CARD).pack(anchor="w", pady=(10,5))
        pm_combo = combo(form, ["Cash", "Debit Card", "Credit Card"], v_pay_method); pm_combo.pack(fill="x")
        
        card_frame = tk.Frame(form, bg=BG_CARD); card_frame.pack(fill="x")

        # Step 4: Confirm
        gold_btn(form, "Confirm Booking", do_smart_book, width=22).pack(pady=30, fill="x")

    def _checkout_dialog(self):
            sel = self.tv.selection()
            if not sel: return
    
            bid = self.tv.item(sel[0])['values'][0]
    
            if messagebox.askyesno("Checkout", "Confirm Payment and Checkout?"):
              conn = get_conn(); cur = conn.cursor()
        
             # 1. Update Booking Status
            cur.execute("UPDATE bookings SET status='Checked-Out' WHERE booking_id=?", (bid,))
        
            # 2. Update Bill and Date in ONE query (Saves processing time)
            cur.execute("""
            UPDATE bills 
            SET paid='Yes', payment_date=CURRENT_DATE 
            WHERE booking_id=?
        """, (bid,))
        
            # 3. Release Room and set to Dirty
            cur.execute("""
            UPDATE rooms SET status='Available', hk_status='Dirty' 
            WHERE room_no=(SELECT room_no FROM bookings WHERE booking_id=?)
        """, (bid,))
        
            conn.commit(); conn.close()
            messagebox.showinfo("Success", "Payment confirmed and guest checked out.")
            self.refresh()

    def _cancel_booking(self):
        sel = self.tv.selection()
        if not sel: return
        bid = self.tv.item(sel[0])['values'][0]
        
        if messagebox.askyesno("Cancel", "Confirm Cancellation?"):
            conn = get_conn(); cur = conn.cursor()
            
            # Update booking
            cur.execute("UPDATE bookings SET status='Cancelled' WHERE booking_id=?", (bid,))
            
            # DELETE the bill because it's no longer valid
            cur.execute("DELETE FROM bills WHERE booking_id=?", (bid,))
            
            conn.commit(); conn.close()
            self.refresh()


    def refresh(self):
        self.tv.delete(*self.tv.get_children())
        conn = get_conn(); cur = conn.cursor()
        
        # Ensure 'paid' is part of the SELECT query
        cur.execute("""
            SELECT b.bill_id, c.name, b.booking_id, b.total_amt, b.paid 
            FROM bills b 
            JOIN bookings bk ON b.booking_id = bk.booking_id
            JOIN customers c ON bk.customer_id = c.customer_id
        """)
        
        for row in cur.fetchall():
            # Format currency and insert into table
            amt = f"₹{row[3]:,.2f}"
            self.tv.insert("", "end", values=(row[0], row[1], row[2], amt, row[4]))
        conn.close()

   

# ─────────────────────────────────────────
# BILLING PAGE
# ─────────────────────────────────────────

class BillingPage(tk.Frame):
    def __init__(self, parent, app):
        super().__init__(parent, bg=BG_DARK)
        self.app = app
        self._build()

    def _build(self):
        # Header Area
        hdr = tk.Frame(self, bg=BG_DARK, pady=20, padx=30)
        hdr.pack(fill="x")
        tk.Label(hdr, text="Billing System", font=FONT_TITLE, fg=ACCENT, bg=BG_DARK).pack(side="left")
        
        # --- NEW: Action Button for Analytics ---
        gold_btn(hdr, "📈 Record Expense", self.add_expense_dialog).pack(side="right", padx=5)

        body = tk.Frame(self, bg=BG_DARK)
        body.pack(fill="both", expand=True, padx=30)

        # Left Column: Bills Table
        left = tk.Frame(body, bg=BG_DARK)
        left.pack(side="left", fill="both", expand=True, padx=(0,20))
        section_title(left, "All Bills").pack(fill="x", pady=(0,10))
        
        cols = ["Bill ID", "Guest", "Booking ID", "Amount", "Paid"]
        widths = [70, 160, 90, 110, 80]
        tv_frame, self.tv = build_treeview(left, cols, widths)
        tv_frame.pack(fill="both", expand=True)

        # Right Column: Invoice Panel
        right = tk.Frame(body, bg=BG_CARD, width=320, padx=20, pady=20,
                         highlightthickness=1, highlightbackground=BORDER)
        right.pack(side="right", fill="y")
        right.pack_propagate(False)

        tk.Label(right, text="Generate Invoice", font=FONT_HEADER, fg=ACCENT, bg=BG_CARD).pack(anchor="w", pady=(0,14))
        self.v_bid = tk.StringVar()
        
        tk.Label(right, text="Booking ID", font=FONT_SMALL, fg=TEXT_MUTED, bg=BG_CARD).pack(anchor="w", pady=(0,4))
        entry(right, textvariable=self.v_bid).pack(anchor="w", fill="x")
        
        gold_btn(right, "🧾 Generate Bill", self._generate_bill, width=22).pack(pady=12, fill="x")

        # Invoice display area
        self.inv_frame = tk.Frame(right, bg=BG_DARK, padx=12, pady=12)
        self.inv_frame.pack(fill="both", expand=True)
        self.inv_lbl = tk.Label(self.inv_frame, text="Select a booking ID\nand generate invoice.",
                                font=FONT_SMALL, fg=TEXT_MUTED, bg=BG_DARK, justify="left")
        self.inv_lbl.pack(anchor="w")

       # 1. Define the function first
       # Define the function at the end of _build
        def on_click(_): 
            sel = self.tv.selection()
            if sel:
                self.v_bid.set(self.tv.item(sel[0])['values'][2])

        self.tv.bind("<Double-1>", on_click)

    def refresh(self):
        self.tv.delete(*self.tv.get_children())
        conn = get_conn(); cur = conn.cursor()
        cur.execute("""SELECT bl.bill_id, c.name, bl.booking_id, bl.total_amt, bl.paid, bl.payment_date
                       FROM bills bl 
                       JOIN bookings b ON bl.booking_id=b.booking_id
                       JOIN customers c ON b.customer_id=c.customer_id
                       ORDER BY bl.bill_id DESC""")
        
        for row in cur.fetchall():
            tag = "paid" if row[4] == "Yes" else "pend"
            self.tv.insert("", "end", values=(row[0], row[1], row[2], f"₹{row[3]:,.2f}", row[4]), tags=(tag,))
                       
        self.tv.tag_configure("paid", background="#152218", foreground=SUCCESS)
        self.tv.tag_configure("pend", background="#2E2A12", foreground=ACCENT)
        conn.close()

    def add_expense_dialog(self):
        win = tk.Toplevel(self)
        win.title("Record Hotel Expense")
        win.geometry("400x350")
        win.configure(bg=BG_CARD)
        win.grab_set()

        v_cat = tk.StringVar(); v_amt = tk.StringVar()

        tk.Label(win, text="Expense Category", bg=BG_CARD, fg=TEXT_MUTED).pack(pady=(20,5))
        entry(win, textvariable=v_cat).pack()
        tk.Label(win, text="Amount (₹)", bg=BG_CARD, fg=TEXT_MUTED).pack(pady=(20,5))
        entry(win, textvariable=v_amt).pack()

        def save():
            try:
                amt = float(v_amt.get())
                conn = get_conn(); cur = conn.cursor()
                cur.execute("INSERT INTO expenses (category, amount) VALUES (?,?)", (v_cat.get(), amt))
                conn.commit(); conn.close()
                messagebox.showinfo("Success", "Expense recorded for monthly analysis.")
                win.destroy()
            except:
                messagebox.showerror("Error", "Invalid Amount")

        gold_btn(win, "Save Expense", save).pack(pady=30)

    # --- Add Expense & Generate Bill methods follow here ---
        

    def _generate_bill(self):
        try:
            bid = int(self.v_bid.get())
        except ValueError:
            messagebox.showerror("Error", "Enter a valid Booking ID.")
            return
        conn = get_conn(); cur = conn.cursor()
        cur.execute("""SELECT b.booking_id, c.name, c.phone, c.country, c.meal_plan,
            r.room_no, r.room_type, r.price, b.check_in, b.check_out,
            b.adults, b.children, b.status
            FROM bookings b JOIN customers c ON b.customer_id=c.customer_id
            JOIN rooms r ON b.room_no=r.room_no
            WHERE b.booking_id=?""", (bid,))
        row = cur.fetchone()
        if not row:
            messagebox.showerror("Not Found", f"No booking found with ID {bid}.")
            conn.close(); return
        _, name, phone, country, meal, room_no, room_type, price, ci, co, adults, children, status = row
        nights     = max((datetime.date.fromisoformat(co) - datetime.date.fromisoformat(ci)).days, 1)
        # Ensure the invoice displays the correct tiered pricing
        room_chg = calculate_stay_price(room_type, nights)
        meal_rate  = {"BB":300,"HB":600,"FB":900}.get(meal, 300)
        meal_chg   = meal_rate * nights * (adults + children)
        tax        = round((room_chg + meal_chg) * 0.12, 2)
        total      = round(room_chg + meal_chg + tax, 2)
        cur.execute("SELECT paid FROM bills WHERE booking_id=?", (bid,))
        br = cur.fetchone()
        paid_status = br[0] if br else "Pending"
        conn.close()

        lines = [
            "━━━━━━━━━━━━━━━━━━━━━━━━━━━",
            "    HOTEL INVOICE",
            "━━━━━━━━━━━━━━━━━━━━━━━━━━━",
            f"Booking ID  : {bid}",
            f"Guest       : {name}",
            f"Phone       : {phone}",
            f"Country     : {country}",
            f"Room        : {room_no} ({room_type})",
            f"Meal Plan   : {meal}",
            f"Check-in    : {ci}",
            f"Check-out   : {co}",
            f"Nights      : {nights}",
            f"Adults/Child: {adults}/{children}",
            "───────────────────────────",
            f"Room Charges: ₹{room_chg:,.2f}",
            f"Meal Charges: ₹{meal_chg:,.2f}",
            f"Tax (12%)   : ₹{tax:,.2f}",
            "───────────────────────────",
            f"TOTAL       : ₹{total:,.2f}",
            f"Status      : {paid_status}",
            "━━━━━━━━━━━━━━━━━━━━━━━━━━━",
        ]

        for w in self.inv_frame.winfo_children():
            w.destroy()
        tk.Label(self.inv_frame,
                 text="\n".join(lines),
                 font=("Courier", 9), fg=TEXT_MAIN, bg=BG_DARK,
                 justify="left", anchor="nw").pack(fill="both", expand=True)
        



# ─────────────────────────────────────────
# HOUSEKEEPING PAGE
# ─────────────────────────────────────────
class HousekeepingPage(tk.Frame):
    def __init__(self, parent, app):
        super().__init__(parent, bg=BG_DARK)
        self.app = app
        self._build()

    def _build(self):
        hdr = tk.Frame(self, bg=BG_DARK, pady=20, padx=30)
        hdr.pack(fill="x")
        tk.Label(hdr, text="Housekeeping Management", font=FONT_TITLE, fg=ACCENT, bg=BG_DARK).pack(side="left")

        body = tk.Frame(self, bg=BG_DARK, padx=30)
        body.pack(fill="both", expand=True)

        # Control Buttons
        btn_bar = tk.Frame(body, bg=BG_DARK)
        btn_bar.pack(fill="x", pady=(0,15))
        gold_btn(btn_bar, "✨ Mark Cleaned", lambda: self._update_hk("Clean"), width=15).pack(side="left", padx=5)
        gold_btn(btn_bar, "🧹 Mark Dirty", lambda: self._update_hk("Dirty"), width=15).pack(side="left", padx=5)
        danger_btn(btn_bar, "📞 Guest Call", lambda: self._update_hk("Service Requested"), width=15).pack(side="left", padx=5)

        # Room Status Table
        cols = ["Room No", "Type", "Occupancy", "HK Status"]
        widths = [80, 120, 120, 180]
        f, self.tv = build_treeview(body, cols, widths)
        f.pack(fill="both", expand=True)

    def refresh(self):
        self.tv.delete(*self.tv.get_children())
        conn = get_conn(); cur = conn.cursor()
        cur.execute("SELECT room_no, room_type, status, hk_status FROM rooms")
        for row in cur.fetchall():
            # Color coding based on status
            tag = "dirty" if row[3] == "Dirty" else "clean"
            if row[3] == "Service Requested": tag = "request"
            self.tv.insert("", "end", values=row, tags=(tag,))
        
        self.tv.tag_configure("dirty", background="#2E1A1A") 
        self.tv.tag_configure("clean", background="#152218") 
        self.tv.tag_configure("request", background="#2E2A12") 
        conn.close()

    def _update_hk(self, new_status):
        sel = self.tv.selection()
        if not sel:
            messagebox.showinfo("Selection", "Please select a room first.")
            return
        room_no = self.tv.item(sel[0])['values'][0]
        conn = get_conn(); cur = conn.cursor()
        cur.execute("UPDATE rooms SET hk_status=? WHERE room_no=?", (new_status, room_no))
        conn.commit(); conn.close()
        self.refresh()
# ─────────────────────────────────────────
# ENTRY POINT
# ─────────────────────────────────────────

if __name__ == "__main__":
    app = HotelApp()
    app.mainloop()
