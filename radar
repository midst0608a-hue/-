import pandas as pd
import time
import requests
import urllib3
import io
import yfinance as yf
import mplfinance as mpf
import os
import re

# 關閉 SSL 驗證警告
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

# --- 基礎配置 ---
CATEGORIES = {"520": "外買", "530": "外賣", "880": "投買", "890": "投賣"}
DAYS = [1, 3, 5, 10, 20]
BASE_URL = "https://fubon-ebrokerdj.fbs.com.tw/z/zk/zk4/zkparse_{}_{}.djhtm"
HEADERS = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'}
PRIORITY_MAP = {"🔴合買": 1, "🟢合賣": 2, "🟠互砍": 3, "🔵新進": 4, "": 5}

def get_full_ticker(stock_raw):
    """提取代號並準備兩種後綴"""
    match = re.search(r'(\d{4,})', stock_raw)
    return match.group(1) if match else None

def fetch_zhuli(day):
    """抓取主力買賣超並排除 00 開頭標的，取前二十名"""
    buy_list = []
    sell_list = []
    for b in [0, 1]:  # 0=上市, 1=上櫃
        url = f"https://5850web.moneydj.com/z/zg/zgk.djhtm?A=F&B={b}&C={day}"
        try:
            resp = requests.get(url, headers=HEADERS, timeout=10, verify=False)
            resp.encoding = 'cp950'
            tables = pd.read_html(io.StringIO(resp.text))
            
            # 【優化】安全獲取表格
            valid_tables = [t for t in tables if t.shape[1] >= 10 and t.to_string().find('名稱') != -1]
            if not valid_tables: continue
            df = valid_tables[0]
            
            header_idx = -1
            for i in range(len(df)):
                if '名稱' in str(df.iloc[i, 1]) or '名稱' in str(df.iloc[i, 6]):
                    header_idx = i
                    break
            
            if header_idx != -1:
                df = df.iloc[header_idx+1:].copy()
                for _, row in df.iterrows():
                    # 買超
                    s_buy = str(row.iloc[1]).replace(' ', '').replace('\u3000', '').replace('\xa0', '').strip()
                    if s_buy and s_buy != 'nan' and not s_buy.startswith('00'):
                        try:
                            v_buy = str(row.iloc[2]).replace(',', '')
                            buy_list.append({'股票名稱': s_buy, '天數': day, '類別': '主力買', '張數': float(v_buy)})
                        except: pass
                    
                    # 賣超
                    s_sell = str(row.iloc[6]).replace(' ', '').replace('\u3000', '').replace('\xa0', '').strip()
                    if s_sell and s_sell != 'nan' and not s_sell.startswith('00'):
                        try:
                            v_sell = str(row.iloc[7]).replace(',', '')
                            sell_list.append({'股票名稱': s_sell, '天數': day, '類別': '主力賣', '張數': -abs(float(v_sell))})
                        except: pass
        except: pass

    # 排序並各取前二十名
    buy_list = sorted(buy_list, key=lambda x: x['張數'], reverse=True)[:20]
    sell_list = sorted(sell_list, key=lambda x: x['張數'], reverse=False)[:20]
    return buy_list + sell_list

def fetch_value_increase():
    """抓取上市/上櫃成交值增排行"""
    val_inc_dict = {}
    for b in [0, 1]:
        url = f"https://5850web.moneydj.com/z/zg/zg_C_{b}_0.djhtm"
        try:
            resp = requests.get(url, headers=HEADERS, timeout=10, verify=False)
            resp.encoding = 'cp950'
            tables = pd.read_html(io.StringIO(resp.text))
            
            valid_tables = [t for t in tables if t.shape[1] >= 8 and t.to_string().find('名稱') != -1]
            if not valid_tables: continue
            df = valid_tables[0]
            
            header_idx = -1
            for i in range(len(df)):
                if '名稱' in str(df.iloc[i, 1]):
                    header_idx = i
                    break
            
            if header_idx != -1:
                df = df.iloc[header_idx+1:].copy()
                for _, row in df.iterrows():
                    name = str(row.iloc[1]).replace(' ', '').replace('\u3000', '').replace('\xa0', '').strip()
                    if name and name != 'nan':
                        rank = str(row.iloc[0]).strip()
                        val_today_str = str(row.iloc[5]).replace(',', '').strip()
                        val_yest_str = str(row.iloc[6]).replace(',', '').strip()
                        
                        try:
                            val_today_yi = float(val_today_str) / 100000
                            val_yest_yi = float(val_yest_str) / 100000
                            v_today = f"{val_today_yi:.2f}"
                            v_yest = f"{val_yest_yi:.2f}"
                        except:
                            v_today = val_today_str
                            v_yest = val_yest_str
                            
                        val_inc_dict[name] = {
                            '今日成交值增': f"第{rank}名",
                            '今日值(億)': v_today,
                            '昨日值(億)': v_yest
                        }
        except: pass
    return val_inc_dict

def fetch_margin_increase():
    """抓取上市/上櫃融資增加排行 (1週)"""
    margin_inc_dict = {}
    for b in [0, 1]:
        url = f"https://5850web.moneydj.com/z/zg/zg_E_{b}_-1.djhtm"
        try:
            resp = requests.get(url, headers=HEADERS, timeout=10, verify=False)
            resp.encoding = 'cp950'
            tables = pd.read_html(io.StringIO(resp.text))
            
            valid_tables = [t for t in tables if t.shape[1] >= 8 and t.to_string().find('名稱') != -1]
            if not valid_tables: continue
            df = valid_tables[0]
            
            header_idx = -1
            for i in range(len(df)):
                if '名稱' in str(df.iloc[i, 1]):
                    header_idx = i
                    break
            
            if header_idx != -1:
                df = df.iloc[header_idx+1:].copy()
                for _, row in df.iterrows():
                    name = str(row.iloc[1]).replace(' ', '').replace('\u3000', '').replace('\xa0', '').strip()
                    if name and name != 'nan':
                        rank = str(row.iloc[0]).strip()
                        yest_bal_str = str(row.iloc[5]).replace(',', '').strip()
                        inc_str = str(row.iloc[7]).replace(',', '').strip()
                        
                        try:
                            yest_bal = float(yest_bal_str)
                            inc_val = float(inc_str)
                            pct = (inc_val / yest_bal) * 100 if yest_bal > 0 else 0
                            pct_str = f"{pct:.2f}%"
                        except:
                            pct_str = ""
                            
                        margin_inc_dict[name] = {
                            '一周融資增加': f"第{rank}名",
                            '融資增加百分比': pct_str
                        }
        except: pass
    return margin_inc_dict

def fetch_macd_positive_tickers():
    """抓取 MACD 選股並過濾 DIF 為正數 (>=0) 的股票，回傳代號集合"""
    macd_tickers = set()
    url = "https://fubon-ebrokerdj.fbs.com.tw/z/zk/zk3/zkparse_740_NA.djhtm"
    try:
        resp = requests.get(url, headers=HEADERS, timeout=10, verify=False)
        resp.encoding = 'big5'
        html_text = resp.text.encode('utf-8', 'ignore').decode('utf-8')
        tables = pd.read_html(io.StringIO(html_text))
        
        for df in tables:
            if df.shape[1] >= 5 and df.to_string().find('DIF') != -1:
                header_idx = -1
                for i in range(len(df)):
                    if 'DIF' in str(df.iloc[i].values):
                        header_idx = i
                        break
                
                if header_idx != -1:
                    data_df = df.iloc[header_idx+1:].copy()
                    for _, row in data_df.iterrows():
                        stock_raw = str(row.iloc[0]).replace(' ', '').replace('\u3000', '').replace('\xa0', '').strip()
                        if stock_raw and stock_raw != 'nan' and '條件' not in stock_raw:
                            try:
                                dif_val = float(str(row.iloc[4]).replace(',', ''))
                                if dif_val >= 0:
                                    ticker = get_full_ticker(stock_raw)
                                    if ticker:
                                        macd_tickers.add(ticker)
                            except: pass
                break
    except Exception as e:
        print(f"⚠️ 抓取 MACD 發生錯誤: {e}")
    return macd_tickers

def save_kline_robust(code):
    """嘗試 .TW 與 .TWO 抓取 K 線 (含 MACD)，並相容新版 yfinance"""
    if not code: return None
    
    # 【關鍵修正】把暫存圖片存到絕對安全的 C:\Temp 裡，避開 OneDrive 干擾
    import os
    temp_dir = r"C:\Temp\籌碼雷達暫存圖"
    if not os.path.exists(temp_dir):
        os.makedirs(temp_dir)
        
    path = os.path.join(temp_dir, f"k_{code}.png")
    
    # 如果有舊圖卡住，先嘗試刪除，確保每次都是最新畫面
    if os.path.exists(path): 
        try:
            os.remove(path)
        except:
            pass 

    for suffix in [".TW", ".TWO"]:
        ticker = code + suffix
        try:
            # 加上 multi_level_index=False 避免新版 yfinance 產生雙層標籤
            df = yf.download(ticker, period="3mo", interval="1d", progress=False, multi_level_index=False)
            
            if df.empty:
                continue

            # 新版 yfinance 雙層索引防呆解除
            if isinstance(df.columns, pd.MultiIndex):
                try:
                    df.columns = df.columns.droplevel(1)
                except: pass
            
            # 確保有 Close 欄位才能畫圖
            if 'Close' not in df.columns:
                print(f"⚠️ 無法畫圖：{ticker} 抓到的資料缺乏 'Close' 欄位！")
                continue
            
            # 剃除 NaN 值，防止 MACD 計算與畫圖崩潰
            df = df.dropna(subset=['Close'])
            
            if len(df) > 26:
                # MACD 計算
                exp1 = df['Close'].ewm(span=12, adjust=False).mean()
                exp2 = df['Close'].ewm(span=26, adjust=False).mean()
                macd = exp1 - exp2
                signal = macd.ewm(span=9, adjust=False).mean()
                hist = macd - signal
                
                # 設定附加圖表
                ap = [
                    mpf.make_addplot(macd, panel=1, color='blue', width=0.7),
                    mpf.make_addplot(signal, panel=1, color='orange', width=0.7),
                    mpf.make_addplot(hist, type='bar', panel=1, color='dimgray')
                ]
                
                mc = mpf.make_marketcolors(up='red', down='green', inherit=True)
                s = mpf.make_mpf_style(marketcolors=mc, gridstyle='--')
                mpf.plot(df, type='candle', addplot=ap, style=s, panel_ratios=(2, 1),
                         savefig=dict(fname=path, dpi=55, bbox_inches='tight'), 
                         axisoff=True, figsize=(3.0, 1.9))
                return path
            else:
                print(f"⚠️ {ticker} 交易天數不足 26 天 ({len(df)}天)，無法計算 MACD。")
        except Exception as e:
            # 印出真正的錯誤，不再默默跳過
            print(f"❌ 畫 {ticker} 的 K 線圖時發生錯誤：{e}")
            continue
            
    return None

def run_v8_3_radar():
    print("啟動 v8.3 修正版：解決排序、K線顯示與防呆崩潰問題...")
    all_raw = []
    kline_cache = {}

    # 1. 抓取數據
    for d in DAYS:
        for code, label in CATEGORIES.items():
            url = BASE_URL.format(code, d)
            try:
                resp = requests.get(url, headers=HEADERS, timeout=10, verify=False)
                resp.encoding = 'cp950'
                tables = pd.read_html(io.StringIO(resp.text))
                
                # 【優化】安全獲取表格
                valid_tables = [t for t in tables if t.shape[1] >= 4 and t.to_string().find('名稱') != -1]
                if not valid_tables: continue
                df = valid_tables[0]
                
                header_idx = 0
                for i in range(len(df)):
                    if '名稱' in str(df.iloc[i, 0]):
                        header_idx = i
                        break
                df.columns = df.iloc[header_idx]; df = df[header_idx+1:].copy()
                n_col, v_col = df.columns[0], [c for c in df.columns if '超' in str(c)][0]
                for _, row in df.iterrows():
                    stock = str(row[n_col]).replace(' ', '').replace('\u3000', '').replace('\xa0', '').strip()
                    if 'nan' in stock or not stock: continue
                    vol = float(str(row[v_col]).replace(',', ''))
                    if label in ["外賣", "投賣"]: vol = -abs(vol)
                    all_raw.append({'股票名稱': stock, '天數': d, '類別': label, '張數': vol})
                time.sleep(0.2)
            except: pass
            
        # 抓取主力買賣超，自動排除 00 開頭標的並取前二十名
        raw_zhuli = fetch_zhuli(d)
        all_raw.extend(raw_zhuli)
        time.sleep(0.2)
        
    raw_df = pd.DataFrame(all_raw)
    
    # 整合買賣動態至單一欄位 (因為買是正數、賣是負數，合併後自動變成淨動向)
    if not raw_df.empty:
        cat_map = {"外買": "外資動向", "外賣": "外資動向", "投買": "投信動向", "投賣": "投信動向", "主力買": "主力動向", "主力賣": "主力動向"}
        raw_df['類別'] = raw_df['類別'].map(cat_map)
    # 【修正 1】：攔截完全沒抓到資料的致命崩潰
    if raw_df.empty:
        print("❌ 錯誤：無法從網站抓取任何資料！(可能是遇到非交易日，或 IP 暫時遭到網站阻擋)。")
        return

    import datetime
    import os
    
    # 建立一個安全的本機資料夾來存放報表（避開 OneDrive）
    save_dir = r"C:\Temp\籌碼雷達報表"
    if not os.path.exists(save_dir):
        os.makedirs(save_dir)  # 如果資料夾不存在，就自動建立一個
        
    # 取得現在時間，產生不重複檔名
    timestamp = datetime.datetime.now().strftime("%m%d_%H%M%S")
    file_name = f"籌碼雷達_終極修復版_{timestamp}.xlsx"
    
    # 組合出完整的絕對路徑
    full_path = os.path.join(save_dir, file_name)
    
    writer = pd.ExcelWriter(full_path, engine='xlsxwriter')
    print(f"準備寫入檔案：{full_path}")
    
    # 👇 就是缺了下面這最關鍵的一行！我把它補回來了
    workbook = writer.book
    
    # 格式定義
    b_fmt = {'valign': 'vcenter', 'font_size': 12}
    fmts = {"🔴": workbook.add_format(dict(b_fmt, bg_color='#FFC7CE')), 
            "🟢": workbook.add_format(dict(b_fmt, bg_color='#C6EFCE')),
            "🟠": workbook.add_format(dict(b_fmt, bg_color='#FFEB9C')), 
            "🔵": workbook.add_format(dict(b_fmt, bg_color='#DDEBF7'))}

    # 2. 處理「總覽看板」
    pivot_all = raw_df.pivot_table(index='股票名稱', columns=['天數', '類別'], values='張數', aggfunc='sum').fillna(0)
    pivot_all.columns = [f"{d}D_{l}" for d, l in pivot_all.columns]
    pivot_all.reset_index(inplace=True)

    # 預先抓取附加資料
    macd_positive_tickers = fetch_macd_positive_tickers()
    val_inc_data = fetch_value_increase()
    margin_inc_data = fetch_margin_increase()

    def get_main_sig(r):
        ticker = get_full_ticker(r['股票名稱'])
        is_macd = ticker in macd_positive_tickers

        f1, t1 = r.get('1D_外資動向', 0), r.get('1D_投信動向', 0)
        sig = ""
        if f1 > 0 and t1 > 0: sig = "🔴合買"
        elif f1 < 0 and t1 < 0: sig = "🟢合賣"
        elif (f1 > 0 and t1 < 0) or (f1 < 0 and t1 > 0): sig = "🟠互砍"
        else:
            lt = [c for c in r.index if any(x in str(c) for x in ["3D", "5D", "10D", "20D"]) and any(y in str(c) for y in ["外資動向", "投信動向"])]
            if (f1 != 0 or t1 != 0) and r[lt].abs().sum() == 0: sig = "🔵新進"

        if is_macd:
            sig = sig + "[MACD]" if sig else "[MACD]"
            
        return sig

    pivot_all.insert(1, '即時訊號', pivot_all.apply(get_main_sig, axis=1))
    pivot_all['P'] = pivot_all['即時訊號'].map(lambda x: PRIORITY_MAP.get(str(x).replace('[MACD]', ''), 5))
    pivot_all = pivot_all.sort_values(['P', '股票名稱']).drop(columns=['P'])

    # 標記無訊號但屬單買賣前三的股票，確保也畫圖
    top_stocks = set()
    for col in ['1D_外資動向', '1D_投信動向']:
        if col in pivot_all.columns:
            top_stocks.update(pivot_all.sort_values(by=col, ascending=False).head(3)['股票名稱'])
            top_stocks.update(pivot_all.sort_values(by=col, ascending=True).head(3)['股票名稱'])
            
    # 主力買賣超前二十名也標記，確保畫出K線
    for col in ['1D_主力動向']:
        if col in pivot_all.columns:
            top_stocks.update(pivot_all.sort_values(by=col, ascending=False).head(20)['股票名稱'])
            top_stocks.update(pivot_all.sort_values(by=col, ascending=True).head(20)['股票名稱'])

    # 3. 處理「各天數明細」
    day_data = {}
    for d in DAYS:
        filtered_df = raw_df[raw_df['天數'] == d]
        
        # 【修正 2】：攔截特定天數沒資料的崩潰
        if filtered_df.empty:
            print(f"⚠️ 警告：找不到 {d} 天期的資料，跳過處理。")
            empty_cols = ['股票名稱', '本日訊號']
            if d == 1: empty_cols.extend(['今日成交值增', '今日值(億)', '昨日值(億)', '一周融資增加', '融資增加百分比'])
            day_data[d] = pd.DataFrame(columns=empty_cols)
            continue

        df_d = filtered_df.pivot_table(index='股票名稱', columns='類別', values='張數', aggfunc='sum').fillna(0).reset_index()
        def d_sig(r):
            ticker = get_full_ticker(r['股票名稱'])
            is_macd = ticker in macd_positive_tickers

            fv, tv = r.get('外資動向', 0), r.get('投信動向', 0)
            sig = ""
            if fv > 0 and tv > 0: sig = "🔴合買"
            elif fv < 0 and tv < 0: sig = "🟢合賣"
            elif (fv > 0 and tv < 0) or (fv < 0 and tv > 0): sig = "🟠互砍"
            
            if is_macd:
                sig = sig + "[MACD]" if sig else "[MACD]"
            
            return sig
        df_d.insert(1, '本日訊號', df_d.apply(d_sig, axis=1))
        df_d['P'] = df_d['本日訊號'].map(lambda x: PRIORITY_MAP.get(str(x).replace('[MACD]', ''), 5))
        df_d = df_d.sort_values(['P', '股票名稱']).drop(columns=['P'])
        
        # 如果是 1D 分頁，加入成交值增註記
        if d == 1:
            df_d['今日成交值增'] = df_d['股票名稱'].map(lambda x: val_inc_data.get(x, {}).get('今日成交值增', ''))
            df_d['今日值(億)'] = df_d['股票名稱'].map(lambda x: val_inc_data.get(x, {}).get('今日值(億)', ''))
            df_d['昨日值(億)'] = df_d['股票名稱'].map(lambda x: val_inc_data.get(x, {}).get('昨日值(億)', ''))
            
            df_d['一周融資增加'] = df_d['股票名稱'].map(lambda x: margin_inc_data.get(x, {}).get('一周融資增加', ''))
            df_d['融資增加百分比'] = df_d['股票名稱'].map(lambda x: margin_inc_data.get(x, {}).get('融資增加百分比', ''))
            
        day_data[d] = df_d

    # --- 視覺化輸出迴圈 ---
    sheets = [('總覽看板', pivot_all, '即時訊號')] + [(f'{d}天明細', day_data[d], '本日訊號') for d in DAYS if not day_data[d].empty]
    
    for s_name, df, sig_col in sheets:
        df.to_excel(writer, sheet_name=s_name, index=False)
        ws = writer.sheets[s_name]
        ws.set_default_row(65) # 稍微加高以容納 MACD
        ws.set_column('A:A', 15) # 恢復正常股票名稱寬度
        
        # 建立專屬放 K 線的右側欄位
        img_col = len(df.columns)
        ws.write(0, img_col, "K線圖走勢")
        ws.set_column(img_col, img_col, 20)

        for enum_idx, (_, row) in enumerate(df.iterrows()):
            sig = row.get(sig_col, "")
            s_name_raw = row.get('股票名稱', "")
            
            if not sig and s_name_raw not in top_stocks: continue
            
            # 1. 強制著色或調整高度
            if sig and sig[0] in fmts: 
                ws.set_row(enum_idx + 1, 65, fmts[sig[0]])
            else:
                ws.set_row(enum_idx + 1, 65)
            
            # 2. 插入 K 線
            if s_name_raw not in kline_cache:
                code = get_full_ticker(s_name_raw)
                kline_cache[s_name_raw] = save_kline_robust(code)
            
            img = kline_cache.get(s_name_raw)
            if img:
                ws.insert_image(enum_idx + 1, img_col, img, {'x_offset': 5, 'y_offset': 2, 'x_scale': 0.65, 'y_scale': 0.65})

    writer.close()
    
    # 清理快取圖檔
    for p in set(kline_cache.values()):
        if p and os.path.exists(p): os.remove(p)
    print("修正版完成！請 Refresh 並下載：籌碼雷達_終極修復版.xlsx")

import traceback

if __name__ == "__main__":
    try:
        run_v8_3_radar()
    except Exception as e:
        print("\n" + "="*50)
        print("❌ 程式發生崩潰！捕捉到以下致命錯誤：\n")
        traceback.print_exc()
        print("="*50)
        input("\n按 Enter 鍵結束程式 (這樣視窗才不會馬上閃退)...")
