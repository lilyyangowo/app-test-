import streamlit as st
import pandas as pd

# 設定網頁標題
st.set_page_config(page_title="家庭記帳與自動分帳系統", layout="wide")
st.title("🏠 家庭記帳與自動分帳系統")

# 1. 初始化 Session State (儲存資料)
if 'members' not in st.session_state:
    st.session_state.members = ["爸爸", "媽媽"]
if 'categories' not in st.session_state:
    st.session_state.categories = ["伙食", "居住", "交通", "教育", "娛樂", "其他"]
if 'records' not in st.session_state:
    st.session_state.records = []

# --- 側邊欄：人員與類別管理 ---
with st.sidebar:
    st.header("⚙️ 設定中心")
    
    # 人員管理
    st.subheader("👥 人員設定")
    new_member = st.text_input("新增成員姓名")
    if st.button("新增成員"):
        if new_member and new_member not in st.session_state.members:
            st.session_state.members.append(new_member)
            st.rerun()
    st.write("目前成員：", ", ".join(st.session_state.members))
    
    st.divider()
    
    # 類別管理 (新增功能)
    st.subheader("📂 類別設定")
    new_cat = st.text_input("新增支出類別", placeholder="例如：寵物、醫療")
    if st.button("新增類別"):
        if new_cat and new_cat not in st.session_state.categories:
            st.session_state.categories.append(new_cat)
            st.rerun()
    st.write("目前類別：", ", ".join(st.session_state.categories))

    st.divider()
    
    if st.button("⚠️ 清除所有紀錄"):
        st.session_state.records = []
        st.rerun()

# --- 主要介面：新增消費 ---
st.header("💸 新增一筆消費")
col1, col2, col3 = st.columns(3)

with col1:
    item_name = st.text_input("項目名稱", placeholder="例如：家樂福買菜")
    # 這裡改成讀取 session_state 裡的動態類別
    category = st.selectbox("支出類別", st.session_state.categories)

with col2:
    amount = st.number_input("消費金額", min_value=0.0, step=10.0)
    payer = st.selectbox("誰付的錢？", st.session_state.members)

with col3:
    st.write("誰要分攤？")
    participants = []
    for m in st.session_state.members:
        if st.checkbox(m, value=True, key=f"check_{m}"):
            participants.append(m)

if st.button("儲存紀錄", type="primary"):
    if not item_name:
        st.error("請輸入項目名稱")
    elif not participants:
        st.error("請至少選擇一位分攤者")
    else:
        split_amount = amount / len(participants)
        st.session_state.records.append({
            "項目": item_name,
            "類別": category,
            "總金額": amount,
            "付款人": payer,
            "參與者": ", ".join(participants), # 轉成字串方便表格顯示
            "參與清單": participants,         # 保留原始清單供計算
            "每人分攤": split_amount
        })
        st.success(f"已記錄：{item_name}")
        st.rerun()

# --- 數據展示與結算 ---
if st.session_state.records:
    df = pd.DataFrame(st.session_state.records)
    
    st.divider()
    st.header("📊 本月消費明細")
    # 顯示時隱藏計算用的「參與清單」欄位
    st.dataframe(df.drop(columns=["參與清單"]), use_container_width=True)

    # 1. 大項目統計
    st.subheader("📁 各類別支出總額")
    cat_summary = df.groupby("類別")["總金額"].sum()
    st.bar_chart(cat_summary)

    # 2. 核心結算邏輯
    st.divider()
    st.header("⚖️ 結算結果 (誰該給誰錢)")
    
    balances = {m: 0.0 for m in st.session_state.members}
    for _, row in df.iterrows():
        balances[row["付款人"]] += row["總金額"]
        for p in row["參與清單"]:
            balances[p] -= row["每人分攤"]

    creditors = [[m, bal] for m, bal in balances.items() if bal > 0.01]
    debtors = [[m, -bal] for m, bal in balances.items() if bal < -0.01]

    if not creditors and not debtors:
        st.info("目前的帳目是平的，不需要分錢！")
    else:
        # 使用簡單的貪婪算法計算還款路徑
        for d in debtors:
            for c in creditors:
                if d[1] <= 0: break
                if c[1] <= 0: continue
                transfer = min(d[1], c[1])
                st.info(f"💰 **{d[0]}** 應支付給 **{c[0]}**： **NT$ {transfer:,.0f}**")
                d[1] -= transfer
                c[1] -= transfer
else:
    st.info("目前還沒有任何消費紀錄，請在上方輸入您的第一筆支出。")
