import os
import time
import warnings
import requests
import schedule
import tweepy
import fitz  # PyMuPDF
from datetime import datetime,timedelta

# SSL警告を無視
warnings.filterwarnings("ignore",category=requests.packages.urllib3.exceptions.InsecureRequestWarning)

# Twitter APIの認証情報を環境変数から取得
API_KEY = os.getenv('TWITTER_API_KEY')
API_SECRET_KEY = os.getenv('TWITTER_API_SECRET_KEY')
ACCESS_TOKEN = os.getenv('TWITTER_ACCESS_TOKEN')
ACCESS_TOKEN_SECRET = os.getenv('TWITTER_ACCESS_TOKEN_SECRET')

# TwitterのAPIに認証
auth = tweepy.OAuthHandler(API_KEY,API_SECRET_KEY)
auth.set_access_token(ACCESS_TOKEN,ACCESS_TOKEN_SECRET)
# あらかじめ環境変数に設定しているため、APIキーとアクセストークンを指定する必要はない

# Twitter API にリクエストを送るためのクラス
client = tweepy.Client(
    consumer_key=API_KEY,
    consumer_secret=API_SECRET_KEY,
    access_token=ACCESS_TOKEN,
    access_token_secret=ACCESS_TOKEN_SECRET
)

# 特別メニューの識別に使われる色（黄色）
TARGET_COLOR = 0xFFFF00

# 日付からPDFのURLを特定する関数
def get_pdf_url(date):
    year = f"{(date.year - 2018):02d}"
    month = f"{date.month:02d}"
    day = f"{date.day:02d}"
    return f"https://www.tsuyama-ct.ac.jp/images/hokushinryou/menu/ryoumenu-R{year}{month}{day}.pdf"

# URLのPDFをダウンロードする関数（失敗時はNoneを返す）
def download_pdf(url):
    try:
        response = requests.get(url,verify=False)  # URLからPDFをダウンロード、その際SSL証明書の検証を無効化
        response.raise_for_status()  # エラーがあれば例外を発生させる
        return response.content
    except Exception as e:
        print("PDFのダウンロード失敗:",e)
        return None

# PDFの指定範囲からテキストを抽出する関数（失敗または空の場合はNoneを返す）
def extract_text_from_pdf_area(pdf_data,rect):
    try:
        doc = fitz.open(stream=pdf_data,filetype="pdf")  # pdf_dataを開く、streamで保存せずに開ける
        text = doc[0].get_text("text",clip=rect).strip()  # (1ページ目の)指定範囲(rect)のテキストを取得
        doc.close()  # メモリを解放して閉じる
        return text if text else None
    # 何も取得できなかった場合はNoneを返す
    except Exception as e:
        return None

# PDFの指定範囲に #FFFF00 の黄色が含まれているか判定する関数
def area_contains_color(pdf_data,rect,target_color):
    doc = fitz.open(stream=pdf_data,filetype="pdf")  # (pdf_data)をPDFとして開く
    pix = doc.get_page_pixmap(0,clip=rect)  # (1ページ目の)指定範囲(rect)の画像を取得
    doc.close()  # メモリを解放して閉じる
    width,height = pix.width,pix.height  # 画像の幅と高さを取得
    for y in range(height):  # 画像の各ピクセルについて
        for x in range(width):
            r,g,b = pix.pixel(x,y)[:3]  # RGB値を取得
            if (r,g,b) == target_color:  # RGB値が指定色と一致する場合
                return True  # Trueを返す
    return False  # 一致する色がない場合はFalseを返す

# ツイートする関数(メイン)
def tweet_text(message):
    try:
        client.create_tweet(text=message)
    except Exception:  # 例外が発生した場合
        print("ツイート失敗")

def do_tweet(mode):
    now = datetime.now()  # 現在時刻を取得
    
    # modeに応じた食事の時間帯を設定
    if mode == 3:  # 翌日の朝食
        target_date = now + timedelta(days=1)  # 現在時刻の1日後
        meal = "朝食"
    elif mode == 1:
        target_date = now
        meal = "昼食"
    elif mode == 2:
        target_date = now
        meal = "夕食"
    
    # 献立PDFの取得対象日を決定
    weekday = target_date.weekday()  # 曜日を取得
    monday = target_date - timedelta(days=weekday)  # 月曜日の日付を取得
    pdf_data = download_pdf(get_pdf_url(monday))  # 月曜日のPDFをダウンロード
    # PDFが取得できなかった場合スキップ
    if pdf_data is None:
        return
    
    # 日付を取得
    header_rects = {  # 曜日ごとの日付の範囲
        0: fitz.Rect(210,132.8,315,148.1),
        1: fitz.Rect(350,132.8,455,148.1),
        2: fitz.Rect(490,132.8,595,148.1),
        3: fitz.Rect(630,132.8,735,148.1),
        4: fitz.Rect(770,132.8,875,148.1),
        5: fitz.Rect(910,132.8,1015,148.1),
        6: fitz.Rect(1050,132.8,1155,148.1)
    }
    header_text = extract_text_from_pdf_area(pdf_data,header_rects[weekday])
    # 日付が取得できなかった場合スキップ
    if header_text is None:
        return
    
    # 曜日ごとのメニューのx座標範囲
    menu_x_ranges = {
        0: (195,330),
        1: (335,470),
        2: (475,610),
        3: (615,750),
        4: (755,890),
        5: (895,1030),
        6: (1035,1170)
    }
    x0,x1 = menu_x_ranges[weekday]  # 曜日に応じたメニューのX座標範囲
    
    # 各食事のメニューを抽出
    if meal == "朝食":
        menu_rect = fitz.Rect(x0,270,x1,295)
        menu_text = extract_text_from_pdf_area(pdf_data,menu_rect)
    elif meal == "昼食":
        menu_rect = (fitz.Rect(x0,375,x1,570)
                     if area_contains_color(pdf_data,fitz.Rect(x0,375,x1,400),TARGET_COLOR)
                     else fitz.Rect(x0,375,x1,400))
        menu_text = extract_text_from_pdf_area(pdf_data,menu_rect)
    elif meal == "夕食":
        menu_rect = (fitz.Rect(x0,580,x1,775)
                     if area_contains_color(pdf_data,fitz.Rect(x0,580,x1,605),TARGET_COLOR)
                     else fitz.Rect(x0,580,x1,605))
        menu_text = extract_text_from_pdf_area(pdf_data,menu_rect)
    
    # メニューが取得できなかった場合スキップ
    if menu_text is None:
        return
    
    # カレーなら絵文字を追加
    if "カレー" in menu_text:
        menu_text += "🍛"

    tweet_message = f"{header_text}の{meal}のメニューは\n「{menu_text}」です。"
    tweet_text(tweet_message)

# ツイートの失敗をキャッチして表示する関数
def safe_do_tweet(mode):
    try:
        do_tweet(mode)
    except Exception:
        print("失敗")

# 毎日のスケジュール設定
schedule.every().day.at("9:00").do(lambda: safe_do_tweet(1))  # 昼食
schedule.every().day.at("14:00").do(lambda: safe_do_tweet(2))  # 夕食
schedule.every().day.at("22:00").do(lambda: safe_do_tweet(3))  # 翌日の朝食
# do()の中は値を直接(または返り値だけ)渡せないので、lambda関数を使って無名の関数を作成して渡す

# スケジュールのループ実行
while True:
    schedule.run_pending()
    time.sleep(1)# twitter-bot
