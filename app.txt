import streamlit as st
import pandas as pd
import sqlite3
import pytesseract
from PIL import Image

# ---------- ฟังก์ชันฐานข้อมูล ----------
def init_db():
    conn = sqlite3.connect("members.db")
    c = conn.cursor()
    c.execute("""
        CREATE TABLE IF NOT EXISTS members (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            no TEXT,
            date TEXT,
            line_name TEXT,
            nickname TEXT,
            group_name TEXT,
            answer TEXT,
            no_answer TEXT,
            join_group7 TEXT,
            join_group8 TEXT,
            newbie TEXT,
            trade TEXT,
            note TEXT,
            reg1 TEXT,
            reg2 TEXT,
            reg3 TEXT
        )
    """)
    conn.commit()
    return conn

def insert_member(conn, row):
    conn.execute("""
        INSERT INTO members
        (no, date, line_name, nickname, group_name, answer, no_answer, join_group7, join_group8, newbie, trade, note, reg1, reg2, reg3)
        VALUES (?,?,?,?,?,?,?,?,?,?,?,?,?,?,?)
    """, tuple(row))
    conn.commit()

def get_all_members(conn):
    return pd.read_sql("SELECT * FROM members", conn)

# ---------- ฟังก์ชัน Parse OCR ----------
def parse_table_from_text(text):
    lines = text.strip().split("\n")
    data = []
    for line in lines:
        parts = line.split("\t")  # แยกตามแท็บ OCR
        if len(parts) >= 15:
            data.append(parts[:15])
    return pd.DataFrame(data, columns=[
        "ลำดับ", "วันที่", "ชื่อไลน์", "ชื่อเล่น", "ชื่อในกลุ่ม",
        "ตอบ", "ไม่ตอบ", "เข้ากลุ่ม7", "เข้ากลุ่ม8", "มือใหม่",
        "เทรดหุ้น", "หมายเหตุ", "ลงทะเบียน1", "ลงทะเบียน2", "ลงทะเบียน3"
    ])

# ---------- เริ่ม Streamlit ----------
st.set_page_config(page_title="ระบบสมาชิก", layout="wide")
st.title("📋 ระบบจัดการข้อมูลสมาชิก")

conn = init_db()

menu = ["อัปโหลดตาราง", "ดูข้อมูล/ค้นหา"]
choice = st.sidebar.selectbox("เมนู", menu)

if choice == "อัปโหลดตาราง":
    st.subheader("อัปโหลดไฟล์ภาพตารางสมาชิก")
    uploaded_file = st.file_uploader("เลือกรูปภาพ", type=["png", "jpg", "jpeg"])

    if uploaded_file:
        image = Image.open(uploaded_file)
        text = pytesseract.image_to_string(image, lang="tha+eng")

        st.text_area("ข้อมูลที่ OCR ได้", text, height=200)

        df = parse_table_from_text(text)
        st.write("✅ ข้อมูลที่ได้จากภาพ:")
        st.dataframe(df)

        if st.button("บันทึกลงฐานข้อมูล"):
            for _, row in df.iterrows():
                insert_member(conn, row)
            st.success("บันทึกข้อมูลลงฐานข้อมูลเรียบร้อย ✅")

elif choice == "ดูข้อมูล/ค้นหา":
    st.subheader("ค้นหาข้อมูลสมาชิก")
    df = get_all_members(conn)

    keyword = st.text_input("🔍 ค้นหาชื่อ, ไลน์, กลุ่ม, วันที่")
    if keyword:
        df = df[df.apply(lambda row: keyword.lower() in str(row).lower(), axis=1)]

    st.dataframe(df, use_container_width=True)
