import requests
from bs4 import BeautifulSoup
from urllib.parse import urlparse, parse_qs, urljoin, urlencode
import random
import string
import re
import argparse
import time
import esprima
import threading
import queue
from collections import defaultdict

# Default configuration
USER_AGENT = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/91.0.4472.124"
COMMON_PARAMS = ["id", "user", "pass", "token", "redirect", "file", "search", "page"]

# Thread-safe storage
parameters = defaultdict(set)
guessed_parameters = defaultdict(set)
api_endpoints = set()
visited_urls = set()
js_files = set()

# Locks for thread safety
param_lock = threading.Lock()
guessed_lock = threading.Lock()
api_lock = threading.Lock()
visited_lock = threading.Lock()
js_lock = threading.Lock()

# Session setup
session = requests.Session()
session.headers.update({"User-Agent": USER_AGENT})

def parse_arguments():
    parser = argparse.ArgumentParser(description="Web App Parameter Enumeration Tool")
    parser.add_argument("url", help="Target URL (e.g., https://example.com)")
    parser.add_argument("--depth", type=int, default=2, help="Max crawl depth (default: 2)")
    parser.add_argument("--delay", type=float, default=0.5, help="Delay between requests (default: 0.5)")
    parser.add_argument("--threads", type=int, default=5, help="Number of threads (default: 5)")
    parser.add_argument("--scope", help="Scope (e.g., 'example.com,/api')", default="")
    return parser.parse_args()

def is_in_scope(url, scope):
    if not scope:
        return True
    parsed_url = urlparse(url)
    for s in scope.split(","):
        if s.startswith("/"):
            if parsed_url.path.startswith(s):
                return True
        else:
            if s in parsed_url.netloc:
                return True
    return False

def fetch_page(url, method="GET", data=None, json_data=None):
    time.sleep(args.delay)
    try:
        if method == "POST":
            if json_data:
                response = session.post(url, json=json_data, timeout=10)
            else:
                response = session.post(url, data=data, timeout=10)
        else:
            response = session.get(url, timeout=10)
        response.raise_for_status()
        return response
    except requests.RequestException as e:
        print(f"Error fetching {url}: {e}")
        return None

def extract_parameters_from_url(url):
    parsed_url = urlparse(url)
    query_params = parse_qs(parsed_url.query)
    with param_lock:
        for param in query_params.keys():
            parameters[param].add(url)

def extract_parameters_from_forms(html, base_url):
    soup = BeautifulSoup(html, "html.parser")
    for form in soup.find_all("form"):
        action = form.get("action")
        method = form.get("method", "GET").upper()
        form_url = urljoin(base_url, action) if action else base_url
        inputs = form.find_all("input")
        form_params = {i.get("name"): i.get("value", "test") for i in inputs if i.get("name")}
        with param_lock:
            for param in form_params:
                parameters[param].add(form_url)
        if method == "POST" and form_params:
            response = fetch_page(form_url, "POST", data=form_params)
            if response:
                guess_parameters(form_url, "POST", form_params)

def extract_links_and_js(html, base_url):
    soup = BeautifulSoup(html, "html.parser")
    links = set()
    for a_tag in soup.find_all("a", href=True):
        full_url = urljoin(base_url, a_tag["href"])
        if is_same_domain(full_url, base_url) and is_in_scope(full_url, args.scope):
            links.add(full_url)
    for script_tag in soup.find_all("script", src=True):
        js_url = urljoin(base_url, script_tag["src"])
        if is_same_domain(js_url, base_url) and is_in_scope(js_url, args.scope):
            with js_lock:
                js_files.add(js_url)
    return links

def is_same_domain(url, base_url):
    return urlparse(url).netloc == urlparse(base_url).netloc

def is_api_endpoint(url, content_type):
    api_patterns = ["api", "v1", "v2", "json", "rest"]
    return any(pattern in url.lower() for pattern in api_patterns) or "json" in content_type.lower()

def generate_unique_value():
    return "GUESS_" + ''.join(random.choices(string.ascii_letters + string.digits, k=8))

def guess_parameters(url, method="GET", base_data=None):
    guessed_params = {param: generate_unique_value() for param in COMMON_PARAMS}
    if method == "GET":
        parsed_url = urlparse(url)
        original_query = parse_qs(parsed_url.query)
        combined_query = original_query.copy()
        for param, value in guessed_params.items():
            combined_query[param] = [value]
        new_url = parsed_url._replace(query=urlencode(combined_query, doseq=True)).geturl()
        response = fetch_page(new_url)
    else:
        combined_data = base_data.copy() if base_data else {}
        for param, value in guessed_params.items():
            combined_data[param] = value
        if is_api_endpoint(url, ""):
            response = fetch_page(url, "POST", json_data=combined_data)
        else:
            response = fetch_page(url, "POST", data=combined_data)
    if response and response.status_code == 200:
        for param, value in guessed_params.items():
            if value in response.text:
                with guessed_lock:
                    guessed_parameters[param].add(url)

def parse_javascript(js_content, base_url):
    try:
        ast = esprima.parseScript(js_content)
        for node in ast.body:
            if node.type == "VariableDeclaration":
                for decl in node.declarations:
                    if decl.init and decl.init.type == "Literal" and isinstance(decl.init.value, str):
                        value = decl.init.value
                        query_match = re.search(r'\?([^#]+)', value)
                        if query_match:
                            params = parse_qs(query_match.group(1))
                            with param_lock:
                                for param in params:
                                    parameters[param].add(base_url)
                        if re.match(r'/(api|v\d|rest)/', value):
                            with api_lock:
                                api_endpoints.add(urljoin(base_url, value))
    except esprima.Error as e:
        print(f"JS parsing error: {e}")

def crawl(url, depth=0):
    if depth > args.depth:
        return
    with visited_lock:
        if url in visited_urls:
            return
        visited_urls.add(url)
    print(f"Crawling: {url} (Depth: {depth})")
    response = fetch_page(url)
    if not response:
        return
    html = response.text
    content_type = response.headers.get("Content-Type", "")
    if is_api_endpoint(url, content_type):
        with api_lock:
            api_endpoints.add(url)
    extract_parameters_from_url(url)
    extract_parameters_from_forms(html, url)
    links = extract_links_and_js(html, url)
    guess_parameters(url)
    for link in links:
        crawl_queue.put((link, depth + 1))

def worker():
    while True:
        try:
            url, depth = crawl_queue.get(timeout=1)
            crawl(url, depth)
        except queue.Empty:
            break
        finally:
            crawl_queue.task_done()

def process_js_files():
    for js_url in js_files:
        response = fetch_page(js_url)
        if response:
            print(f"Parsing JS: {js_url}")
            parse_javascript(response.text, js_url)

def save_results():
    with open("parameters.txt", "w") as f:
        f.write("Extracted Parameters:\n" + "="*50 + "\n")
        for param, urls in parameters.items():
            f.write(f"Parameter: {param}\nFound in URLs: {', '.join(urls)}\n" + "-"*50 + "\n")
        f.write("\nGuessed Parameters:\n" + "="*50 + "\n")
        for param, urls in guessed_parameters.items():
            f.write(f"Parameter: {param}\nPotentially used in URLs: {', '.join(urls)}\n" + "-"*50 + "\n")
        f.write("\nAPI Endpoints:\n" + "="*50 + "\n")
        for url in api_endpoints:
            f.write(f"{url}\n")
    print("Results saved to 'parameters.txt'")

if __name__ == "__main__":
    args = parse_arguments()
    print(f"Starting enumeration on {args.url} with {args.threads} threads")
    crawl_queue = queue.Queue()
    crawl_queue.put((args.url, 0))
    threads = [threading.Thread(target=worker) for _ in range(args.threads)]
    for t in threads:
        t.start()
    crawl_queue.join()
    for t in threads:
        t.join()
    process_js_files()
    save_results()
    print(f"Found {len(parameters)} parameters, {len(guessed_parameters)} guessed, {len(api_endpoints)} APIs")