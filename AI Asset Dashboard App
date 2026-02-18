import streamlit as st
import pandas as pd
import plotly.express as px
import plotly.graph_objects as go
import json
import io
from datetime import datetime

# --- 1. 頁面設定與 CSS 樣式 ---
st.set_page_config(
    page_title="AI 智能財務診斷儀表板",
    page_icon="📊",
    layout="wide",
    initial_sidebar_state="expanded"
)

# 自定義 CSS 以美化卡片與介面 (模仿原本的馬卡龍色系與現代感)
st.markdown("""
<style>
    /* 全局字體優化 */
    .stApp {
        font-family: 'Inter', 'Noto Sans TC', sans-serif;
        background-color: #f8fafc;
    }
    
    /* 頂部大卡片樣式 */
    div[data-testid="metric-container"] {
        background-color: white;
        border: 1px solid #e2e8f0;
        border-radius: 12px;
        padding: 15px;
        box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
        transition: all 0.3s ease;
    }
    div[data-testid="metric-container"]:hover {
        box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.1);
        border-color: #cbd5e1;
    }
    
    /* 讓標題更顯眼 */
    h1, h2, h3 {
        color: #1e293b;
        font-weight: 700;
    }
    
    /* 強調關鍵數字顏色 */
    .big-asset { color: #2563eb; } /* 藍 */
    .big-liability { color: #dc2626; } /* 紅 */
    .big-net { color: #059669; } /* 綠 */
</style>
""", unsafe_allow_html=True)

# --- 2. 初始化 Session State (資料結構) ---
# 定義所有表格的初始欄位結構
DEFAULT_STRUCTURE = {
    "incomes": pd.DataFrame(columns=["項目", "本薪", "津貼", "扣除", "實領(自動)"]),
    "liabilities": pd.DataFrame(columns=["債務名稱", "剩餘餘額", "月還款", "利率(%)"]),
    "banks": pd.DataFrame(columns=["銀行名稱", "存款餘額", "利率(%)", "預估月息(自動)"]),
    "stocks": pd.DataFrame(columns=["代號/名稱", "股數", "現價", "市值(自動)"]),
    "funds": pd.DataFrame(columns=["基金名稱", "投入本金", "目前市值"]),
    "insurance": pd.DataFrame(columns=["保險名稱", "已繳年期", "保單價值"]),
    "cash": pd.DataFrame(columns=["項目", "金額"]),
    "pension": pd.DataFrame(columns=["項目", "累積金額"]),
}

# 如果是第一次執行，初始化這些空的 DataFrame
for key, df in DEFAULT_STRUCTURE.items():
    if key not in st.session_state:
        st.session_state[key] = df

# --- 3. 側邊欄：資料管理 (匯入/匯出) ---
with st.sidebar:
    st.title("⚙️ 數據管理中心")
    st.markdown("---")
    
    # 匯出功能
    st.subheader("📤 備份資料")
    
    # 將當前所有 DataFrame 打包成一個 JSON 字串
    def convert_to_json():
        export_data = {}
        for k in DEFAULT_STRUCTURE.keys():
            # 將 DataFrame 轉為字典列表，方便 JSON 序列化
            export_data[k] = st.session_state[k].to_dict(orient="records")
        return json.dumps(export_data, ensure_ascii=False, indent=4)

    json_str = convert_to_json()
    file_name = f"Finance_Backup_{datetime.now().strftime('%Y-%m-%d')}.json"
    
    st.download_button(
        label="下載 JSON 備份檔",
        data=json_str,
        file_name=file_name,
        mime="application/json",
        help="下載後請妥善保存，下次可直接還原進度。"
    )

    st.markdown("---")

    # 匯入功能
    st.subheader("📥 還原資料")
    uploaded_file = st.file_uploader("上傳 JSON 檔案", type=["json"])
    
    if uploaded_file is not None:
        try:
            data = json.load(uploaded_file)
            # 將讀取的 JSON 轉回 DataFrame 並存入 session_state
            for k in DEFAULT_STRUCTURE.keys():
                if k in data:
                    st.session_state[k] = pd.DataFrame(data[k])
            st.success("✅ 資料還原成功！")
        except Exception as e:
            st.error(f"❌ 檔案格式錯誤：{e}")

    st.markdown("---")
    st.info("💡 提示：本工具為離線單機版，所有資料僅暫存於您的瀏覽器或下載的檔案中，保障隱私。")

# --- 4. 核心邏輯：即時計算總額 ---
# 為了避免每次運算都重新抓取，我們定義一個計算函式
def calculate_totals():
    # 1. 收入 (實領 = 本薪 + 津貼 - 扣除)
    df_inc = st.session_state["incomes"].copy()
    # 確保數值型態正確，避免字串相加
    for col in ["本薪", "津貼", "扣除"]:
        df_inc[col] = pd.to_numeric(df_inc[col], errors='coerce').fillna(0)
    total_income = (df_inc["本薪"] + df_inc["津貼"] - df_inc["扣除"]).sum()

    # 2. 負債
    df_liab = st.session_state["liabilities"].copy()
    df_liab["剩餘餘額"] = pd.to_numeric(df_liab["剩餘餘額"], errors='coerce').fillna(0)
    df_liab["月還款"] = pd.to_numeric(df_liab["月還款"], errors='coerce').fillna(0)
    total_liabilities = df_liab["剩餘餘額"].sum()
    total_monthly_payment = df_liab["月還款"].sum()

    # 3. 資產類別計算
    # 銀行
    df_bank = st.session_state["banks"].copy()
    df_bank["存款餘額"] = pd.to_numeric(df_bank["存款餘額"], errors='coerce').fillna(0)
    df_bank["利率(%)"] = pd.to_numeric(df_bank["利率(%)"], errors='coerce').fillna(0)
    asset_bank = df_bank["存款餘額"].sum()
    est_interest = (df_bank["存款餘額"] * (df_bank["利率(%)"] / 100) / 12).sum()

    # 股票 (市值 = 股數 * 現價)
    df_stock = st.session_state["stocks"].copy()
    df_stock["股數"] = pd.to_numeric(df_stock["股數"], errors='coerce').fillna(0)
    df_stock["現價"] = pd.to_numeric(df_stock["現價"], errors='coerce').fillna(0)
    asset_stock = (df_stock["股數"] * df_stock["現價"]).sum()

    # 其他資產
    asset_fund = pd.to_numeric(st.session_state["funds"]["目前市值"], errors='coerce').fillna(0).sum()
    asset_ins = pd.to_numeric(st.session_state["insurance"]["保單價值"], errors='coerce').fillna(0).sum()
    asset_cash = pd.to_numeric(st.session_state["cash"]["金額"], errors='coerce').fillna(0).sum()
    asset_pension = pd.to_numeric(st.session_state["pension"]["累積金額"], errors='coerce').fillna(0).sum()

    # 總結
    total_assets = asset_bank + asset_stock + asset_fund + asset_ins + asset_cash + asset_pension
    net_worth = total_assets - total_liabilities
    debt_ratio = (total_liabilities / total_assets * 100) if total_assets > 0 else 0
    free_cash_flow = total_income - total_monthly_payment

    return {
        "income": total_income,
        "liabilities": total_liabilities,
        "monthly_payment": total_monthly_payment,
        "assets": total_assets,
        "net_worth": net_worth,
        "debt_ratio": debt_ratio,
        "interest": est_interest,
        "cash_flow": free_cash_flow,
        "breakdown": {
            "銀行存款": asset_bank,
            "股票投資": asset_stock,
            "基金資產": asset_fund,
            "保險價值": asset_ins,
            "流動現金": asset_cash,
            "勞退專戶": asset_pension
        }
    }

totals = calculate_totals()

# --- 5. 主介面：儀表板總覽 ---
st.title("📊 AI 智能財務診斷儀表板")
st.caption(f"最後更新時間: {datetime.now().strftime('%Y-%m-%d %H:%M')}")

# 第一排：核心大數據
col1, col2, col3 = st.columns(3)
with col1:
    st.metric("總資產 (Total Assets)", f"${totals['assets']:,.0f}", delta="資產總額", delta_color="normal")
with col2:
    st.metric("總負債 (Total Liabilities)", f"${totals['liabilities']:,.0f}", delta=f"-{totals['debt_ratio']:.1f}% 負債比", delta_color="inverse")
with col3:
    st.metric("淨資產 (Net Worth)", f"${totals['net_worth']:,.0f}", delta="身價總計", delta_color="normal")

# 第二排：現金流與細節
c1, c2, c3, c4 = st.columns(4)
c1.metric("實領月薪", f"${totals['income']:,.0f}")
c2.metric("預估月利息", f"${totals['interest']:,.0f}")
c3.metric("月還款總額", f"${totals['monthly_payment']:,.0f}")
c4.metric("自由現金流", f"${totals['cash_flow']:,.0f}")

st.markdown("---")

# --- 6. 圖表分析區 ---
chart_col1, chart_col2 = st.columns([2, 1])

with chart_col1:
    st.subheader("資產分佈概況 (Asset Distribution)")
    # 準備資料給 Plotly
    df_chart = pd.DataFrame(list(totals["breakdown"].items()), columns=["類別", "金額"])
    # 建立條狀圖
    fig_bar = px.bar(
        df_chart, x="類別", y="金額", color="類別",
        text_auto='.2s',
        color_discrete_sequence=px.colors.qualitative.Pastel
    )
    fig_bar.update_layout(showlegend=False, margin=dict(t=0, b=0, l=0, r=0), height=300)
    st.plotly_chart(fig_bar, use_container_width=True)

with chart_col2:
    st.subheader("資產配置 (Portfolio)")
    # 建立圓餅圖
    fig_pie = go.Figure(data=[go.Pie(
        labels=df_chart["類別"], 
        values=df_chart["金額"], 
        hole=.6, # 甜甜圈圖
        marker=dict(colors=px.colors.qualitative.Pastel)
    )])
    fig_pie.update_layout(showlegend=False, margin=dict(t=0, b=0, l=0, r=0), height=300)
    st.plotly_chart(fig_pie, use_container_width=True)

st.markdown("---")

# --- 7. 詳細資料輸入區 (Data Entry) ---
st.subheader("📝 詳細資產配置輸入")
st.info("請直接在下方表格中輸入數據，系統會自動儲存並更新上方圖表。")

tab1, tab2, tab3, tab4 = st.tabs(["💵 收入與負債", "🏦 銀行與現金", "📈 投資組合", "🛡️ 保險與退休"])

# 定義 data_editor 的通用設定
editor_config = {"num_rows": "dynamic", "use_container_width": True}

with tab1:
    c1, c2 = st.columns(2)
    with c1:
        st.markdown("#### 收入明細")
        # 收入表格
        edited_income = st.data_editor(
            st.session_state["incomes"],
            column_config={
                "本薪": st.column_config.NumberColumn(format="$%d"),
                "津貼": st.column_config.NumberColumn(format="$%d"),
                "扣除": st.column_config.NumberColumn(format="$%d"),
                "實領(自動)": st.column_config.NumberColumn(disabled=True, help="自動計算，無需輸入"),
            },
            key="editor_income",
            **editor_config
        )
        if not edited_income.equals(st.session_state["incomes"]):
            st.session_state["incomes"] = edited_income
            st.rerun()

    with c2:
        st.markdown("#### 負債管理")
        edited_liab = st.data_editor(
            st.session_state["liabilities"],
            column_config={
                "剩餘餘額": st.column_config.NumberColumn(format="$%d"),
                "月還款": st.column_config.NumberColumn(format="$%d"),
                "利率(%)": st.column_config.NumberColumn(format="%.2f%%"),
            },
            key="editor_liab",
            **editor_config
        )
        if not edited_liab.equals(st.session_state["liabilities"]):
            st.session_state["liabilities"] = edited_liab
            st.rerun()

with tab2:
    c1, c2 = st.columns(2)
    with c1:
        st.markdown("#### 銀行存款")
        edited_bank = st.data_editor(
            st.session_state["banks"],
            column_config={
                "存款餘額": st.column_config.NumberColumn(format="$%d"),
                "利率(%)": st.column_config.NumberColumn(format="%.2f%%"),
                "預估月息(自動)": st.column_config.NumberColumn(disabled=True),
            },
            key="editor_bank",
            **editor_config
        )
        if not edited_bank.equals(st.session_state["banks"]):
            st.session_state["banks"] = edited_bank
            st.rerun()
            
    with c2:
        st.markdown("#### 流動現金")
        edited_cash = st.data_editor(
            st.session_state["cash"],
            column_config={"金額": st.column_config.NumberColumn(format="$%d")},
            key="editor_cash",
            **editor_config
        )
        if not edited_cash.equals(st.session_state["cash"]):
            st.session_state["cash"] = edited_cash
            st.rerun()

with tab3:
    c1, c2 = st.columns(2)
    with c1:
        st.markdown("#### 股票投資")
        edited_stock = st.data_editor(
            st.session_state["stocks"],
            column_config={
                "股數": st.column_config.NumberColumn(format="%d"),
                "現價": st.column_config.NumberColumn(format="$%.2f"),
                "市值(自動)": st.column_config.NumberColumn(disabled=True),
            },
            key="editor_stock",
            **editor_config
        )
        if not edited_stock.equals(st.session_state["stocks"]):
            st.session_state["stocks"] = edited_stock
            st.rerun()
            
    with c2:
        st.markdown("#### 基金/ETF")
        edited_fund = st.data_editor(
            st.session_state["funds"],
            column_config={
                "投入本金": st.column_config.NumberColumn(format="$%d"),
                "目前市值": st.column_config.NumberColumn(format="$%d"),
            },
            key="editor_fund",
            **editor_config
        )
        if not edited_fund.equals(st.session_state["funds"]):
            st.session_state["funds"] = edited_fund
            st.rerun()

with tab4:
    c1, c2 = st.columns(2)
    with c1:
        st.markdown("#### 保險價值")
        edited_ins = st.data_editor(
            st.session_state["insurance"],
            column_config={"保單價值": st.column_config.NumberColumn(format="$%d")},
            key="editor_ins",
            **editor_config
        )
        if not edited_ins.equals(st.session_state["insurance"]):
            st.session_state["insurance"] = edited_ins
            st.rerun()
            
    with c2:
        st.markdown("#### 勞退專戶")
        edited_pension = st.data_editor(
            st.session_state["pension"],
            column_config={"累積金額": st.column_config.NumberColumn(format="$%d")},
            key="editor_pension",
            **editor_config
        )
        if not edited_pension.equals(st.session_state["pension"]):
            st.session_state["pension"] = edited_pension
            st.rerun()

st.markdown("---")

# --- 8. AI 診斷分析區 ---
st.subheader("🤖 AI 財務診斷報告")

if st.button("開始診斷", type="primary"):
    with st.spinner('AI 正在分析您的財務結構...'):
        import time
        time.sleep(1.5) # 模擬分析時間
        
        ratio = totals['debt_ratio']
        advice = ""
        status_color = "green"
        
        if ratio > 60:
            status_color = "red"
            msg = "⚠️ 警示：負債比過高 (>60%)"
            advice = """
            1. **優先理債**：您的負債比例偏高，建議暫停高風險投資，優先償還高利率債務。
            2. **檢視現金流**：確認手邊現金是否足以支撐 6 個月的生活開銷與還款。
            3. **銀行視角**：此比例在申請新貸款時可能會面臨較嚴格的審查或較差的條件。
            """
        elif ratio > 40:
            status_color = "orange"
            msg = "📝 注意：負債比偏高 (40-60%)"
            advice = """
            1. **正常範圍**：若包含房貸，此比例尚屬可接受範圍。
            2. **收支平衡**：請留意「每月還款額」佔「實領月薪」的比例，建議控制在 1/3 以內，以免影響生活品質。
            3. **緊急預備金**：請確保預備金充足。
            """
        else:
            status_color = "green"
            msg = "✅ 良好：財務結構健康"
            advice = """
            1. **資產增值**：您的資產大於負債，財務體質強健。
            2. **投資建議**：若有閒置資金，可考慮定期定額投入大盤型標的（如 0050, SPY）以抵抗通膨。
            3. **定期檢視**：建議每半年檢視一次保單價值與基金績效。
            """
            
        st.markdown(f"### {msg}")
        st.info(advice)
        
        # 額外的小提示
        if totals['interest'] * 12 > totals['income']:
            st.success("🎉 太棒了！您的被動收入（年）已經超過了一個月的薪水！")
