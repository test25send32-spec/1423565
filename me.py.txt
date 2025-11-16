#!/usr/bin/env python3
# bulk_email_sender.py
# ابزار ارسال ایمیل انبوه با پشتیبانی از چند اکانت و چند نخی
# طراحی شده برای اجرا در Pydroid3 و هاست اشتراکی
# نویسنده: Ethical Hacker (شما)
# فقط برای استفاده مجاز

import smtplib
import ssl
import threading
import time
import os
import csv
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
from datetime import datetime

# ----------------------------
# تنظیمات کلی
# ----------------------------
MAX_RETRIES = 3
TIMEOUT = 10
DELAY_BETWEEN_EMAILS = 1  # ثانیه
MAX_THREADS = 5  # برای اجرا در دستگاه‌های ضعیف (مثل موبایل)
LOG_FILE = "email_log.csv"
DATA_DIR = "sent_data"

# ایجاد پوشه ذخیره لاگ
os.makedirs(DATA_DIR, exist_ok=True)
log_path = os.path.join(DATA_DIR, LOG_FILE)

# فرمت تاریخ
def now():
    return datetime.now().strftime("%Y-%m-%d %H:%M:%S")

# ----------------------------
# خواندن لیست اکانت‌ها از فایل
# فرمت: email,password,smtp_server,port,use_tls
# مثال: user@site.com,p@ss123,mail.site.com,587,True
# ----------------------------
def load_accounts(filename="accounts.txt"):
    accounts = []
    try:
        with open(filename, 'r', encoding='utf-8') as f:
            reader = csv.reader(f)
            for row in reader:
                if len(row) >= 5:
                    email, password, server, port, tls = row[:5]
                    accounts.append({
                        "email": email.strip(),
                        "password": password.strip(),
                        "server": server.strip(),
                        "port": int(port),
                        "tls": tls.strip().lower() == 'true'
                    })
    except FileNotFoundError:
        print(f"[!] فایل {filename} پیدا نشد.")
        exit(1)
    except Exception as e:
        print(f"[!] خطا در خواندن اکانت‌ها: {e}")
        exit(1)
    return accounts

# ----------------------------
# خواندن لیست گیرندگان
# ----------------------------
def load_recipients(filename="recipients.txt"):
    try:
        with open(filename, 'r', encoding='utf-8') as f:
            recipients = [line.strip() for line in f if line.strip() and "@" in line]
        return recipients
    except FileNotFoundError:
        print(f"[!] فایل {recipients.txt} پیدا نشد.")
        exit(1)

# ----------------------------
# ساخت ایمیل (HTML + Text)
# ----------------------------
def create_email(sender, recipient, subject, body_text, body_html):
    msg = MIMEMultipart("alternative")
    msg["From"] = sender
    msg["To"] = recipient
    msg["Subject"] = subject

    part1 = MIMEText(body_text, "plain", "utf-8")
    part2 = MIMEText(body_html, "html", "utf-8")

    msg.attach(part1)
    msg.attach(part2)
    return msg.as_string()

# ----------------------------
# ارسال ایمیل از یک اکانت
# ----------------------------
def send_from_account(account, recipients, email_content, lock, sent_count):
    server = account["server"]
    port = account["port"]
    email = account["email"]
    password = account["password"]
    use_tls = account["tls"]

    success_count = 0
    context = ssl.create_default_context() if use_tls else None

    try:
        # اتصال به سرور
        smtp_conn = smtplib.SMTP(server, port, timeout=TIMEOUT)
        if use_tls:
            smtp_conn.starttls(context=context)
        smtp_conn.login(email, password)
        print(f"[+] ✅ ورود موفق به {email}")

        for recipient in recipients:
            try:
                msg = create_email(email, recipient, email_content["subject"],
                                   email_content["text"], email_content["html"])
                smtp_conn.sendmail(email, recipient, msg)
                success_count += 1
                with lock:
                    print(f"[→] ارسال شد به: {recipient} از طریق {email}")
                    sent_count[0] += 1
                    # ذخیره در لاگ
                    with open(log_path, 'a', newline='', encoding='utf-8') as logf:
                        writer = csv.writer(logf)
                        writer.writerow([now(), email, recipient, "Success", server])
                time.sleep(DELAY_BETWEEN_EMAILS)
            except Exception as e:
                with lock:
                    print(f"[!] خطا در ارسال به {recipient}: {str(e)}")
                    with open(log_path, 'a', newline='', encoding='utf-8') as logf:
                        writer = csv.writer(logf)
                        writer.writerow([now(), email, recipient, f"Failed: {e}", server])

        smtp_conn.quit()
        print(f"[✓] {success_count} ایمیل از {email} ارسال شد.")

    except Exception as e:
        with lock:
            print(f"[!] ❌ اتصال به {email} با خطا مواجه شد: {e}")
            with open(log_path, 'a', newline='', encoding='utf-8') as logf:
                writer = csv.writer(logf)
                writer.writerow([now(), email, "", f"Login Failed: {e}", server])

# ----------------------------
# تابع اصلی
# ----------------------------
def main():
    print(f"[+] شروع ارسال ایمیل انبوه در {now()}")

    # خواندن داده‌ها
    accounts = load_accounts()
    recipients = load_recipients()

    if not accounts:
        print("[!] هیچ اکانتی بارگذاری نشد.")
        return
    if not recipients:
        print("[!] هیچ گیرنده‌ای وجود ندارد.")
        return

    print(f"[✓] {len(accounts)} اکانت و {len(recipients)} گیرنده بارگذاری شد.")

    # تنظیم محتوای ایمیل
    subject = "تست ارسال ایمیل انبوه"
    body_text = "این یک پیام تستی است که در حین ارزیابی امنیتی ارسال شده است."
    body_html = """
    <html>
    <body dir="rtl">
        <h3>این یک ایمیل تستی است</h3>
        <p>این پیام در حین تست ارسال انبوه ایمیل ارسال شده است.</p>
        <p><small>ارسال شده در: """ + now() + """</small></p>
    </body>
    </html>
    """

    email_content = {
        "subject": subject,
        "text": body_text,
        "html": body_html
    }

    # رشته‌ها و متغیرهای اشتراکی
    lock = threading.Lock()
    sent_count = [0]  # متغیر قابل دسترسی در ترد
    threads = []

    # ایجاد ترد برای هر اکانت
    for account in accounts:
        time.sleep(1)  # کاهش فشار اولیه
        if len(threads) >= MAX_THREADS:
            break
        thread = threading.Thread(target=send_from_account, args=(
            account, recipients, email_content, lock, sent_count
        ))
        thread.start()
        threads.append(thread)

    # انتظار برای اتمام تردها
    for t in threads:
        t.join()

    print(f"[+] ✅ ارسال کامل شد. تعداد کل ایمیل‌های ارسالی: {sent_count[0]}")
    print(f"[+] لاگ در مسیر ذخیره شد: {log_path}")

# ----------------------------
# ایجاد فایل‌های نمونه (در صورت نبود)
# ----------------------------
def create_sample_files():
    if not os.path.exists("accounts.txt"):
        with open("accounts.txt", "w", encoding="utf-8") as f:
            f.write("your@domain.com,yourpass,mail.domain.com,587,True\n")
            f.write("another@site.com,pass123,smtp.site.com,465,True\n")
        print("[*] فایل accounts.txt نمونه ایجاد شد.")

    if not os.path.exists("recipients.txt"):
        with open("recipients.txt", "w", encoding="utf-8") as f:
            f.write("test1@receiver.com\n")
            f.write("test2@receiver.com\n")
        print("[*] فایل recipients.txt نمونه ایجاد شد.")

    if not os.path.exists(log_path):
        with open(log_path, "w", encoding="utf-8", newline="") as f:
            writer = csv.writer(f)
            writer.writerow(["Time", "Sender", "Recipient", "Status", "SMTP_Server"])

# ----------------------------
# اجرا
# ----------------------------
if __name__ == "__main__":
    create_sample_files()
    main()