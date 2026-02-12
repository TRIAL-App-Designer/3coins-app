import streamlit as st
import pandas as pd
import time
from bs4 import BeautifulSoup
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
import io

# --- ページ設定 ---
st.set_page_config(page_title="3COINSレビュー収集", layout="centered")
st.title("🛍️ 3COINSレビュー収集アプリ")
st.write("商品ページのURLを入力すると、全レビューをExcelでダウンロードできます。")

# --- ブラウザ設定関数 ---
def get_driver():
    options = Options()
    options.add_argument('--headless')
    options.add_argument('--no-sandbox')
    options.add_argument('--disable-dev-shm-usage')
    options.add_argument('--disable-gpu')
    # Streamlit Cloud上でのChromeドライバ設定
    return webdriver.Chrome(options=options)

# --- 入力フォーム ---
target_url = st.text_input("商品URLをここに貼り付け:", placeholder="https://www.palcloset.jp/...")

# --- 実行ボタン ---
if st.button("🚀 収集開始"):
    if not target_url:
        st.error("URLを入力してください！")
    else:
        st.info("ブラウザを起動中...これには数分かかる場合があります。")
        status_text = st.empty() # 進捗表示用
        progress_bar = st.progress(0)
        
        driver = None
        try:
            driver = get_driver()
            driver.get(target_url)
            status_text.text("ページにアクセス中...")
            time.sleep(2)

            # 1. レビュータブクリック
            try:
                status_text.text("レビュータブを開いています...")
                tab_btn = WebDriverWait(driver, 10).until(
                    EC.presence_of_element_located((By.ID, "review_tab"))
                )
                driver.execute_script("arguments[0].click();", tab_btn)
                time.sleep(2)
            except:
                st.warning("レビュータブが見つかりませんでした（既に開いている可能性があります）")

            # 2. 全件表示ループ
            status_text.text("レビューを展開中...")
            loop_count = 0
            while loop_count < 50: # 最大ループ数
                try:
                    more_button = WebDriverWait(driver, 2).until(
                        EC.presence_of_element_located((By.CSS_SELECTOR, "div.more_btn span, button[aria-hidden='true']"))
                    )
                    driver.execute_script("arguments[0].click();", more_button)
                    time.sleep(1.5)
                    loop_count += 1
                    status_text.text(f"レビューを展開中... ({loop_count}回クリック)")
                    progress_bar.progress(min(loop_count * 2, 90)) # 進捗バー更新
                except:
                    break
            
            progress_bar.progress(100)
            status_text.text("データ解析中...")

            # 3. データ抽出
            soup = BeautifulSoup(driver.page_source, "html.parser")
            
            # 各要素取得
            p_colors = soup.select(".review_color")
            p_sizes = soup.select(".review_size")
            p_ages = soup.select(".review_age")
            p_titles = soup.select(".review_title")
            review_places = soup.select(".review_place")
            
            p_bodies = []
            for place in review_places:
                next_p = place.find_next_sibling("p")
                p_bodies.append(next_p.get_text(strip=True) if next_p else "-")

            # データフレーム化
            count = len(p_bodies)
            data = []
            for i in range(count):
                row = {
                    "カラー": p_colors[i].get_text(strip=True) if i < len(p_colors) else "-",
                    "サイズ": p_sizes[i].get_text(strip=True) if i < len(p_sizes) else "-",
                    "年齢": p_ages[i].get_text(strip=True) if i < len(p_ages) else "-",
                    "タイトル": p_titles[i].get_text(strip=True) if i < len(p_titles) else "-",
                    "本文": p_bodies[i]
                }
                data.append(row)

            if not data:
                st.error("データが取得できませんでした。")
            else:
                df = pd.DataFrame(data)
                
                # Excelをメモリ上に保存（ファイルを作らず直接DLさせる技）
                buffer = io.BytesIO()
                with pd.ExcelWriter(buffer, engine='xlsxwriter') as writer:
                    df.to_excel(writer, index=False, sheet_name='Sheet1')
                
                # ダウンロードボタン表示
                st.success(f"🎉 完了！ {len(df)}件のレビューを取得しました。")
                st.download_button(
                    label="📥 Excelファイルをダウンロード",
                    data=buffer.getvalue(),
                    file_name="3coins_reviews.xlsx",
                    mime="application/vnd.ms-excel"
                )

        except Exception as e:
            st.error(f"エラーが発生しました: {e}")
        
        finally:
            if driver:
                driver.quit()