import os
import json
import tkinter as tk
from tkinter import filedialog
from http.server import BaseHTTPRequestHandler, HTTPServer
from urllib.parse import parse_qs, urlparse
from collections import defaultdict

FILE_TYPE_MAP = {
    "图片": {".jpg", ".jpeg", ".png", ".gif", ".bmp", ".svg", ".webp", ".ico"},
    "视频": {".mp4", ".avi", ".mov", ".mkv", ".flv", ".wmv", ".webm"},
    "音频": {".mp3", ".wav", ".flac", ".aac", ".ogg", ".wma"},
    "文档": {".pdf", ".doc", ".docx", ".xls", ".xlsx", ".ppt", ".pptx", ".txt", ".md"},
    "压缩包": {".zip", ".rar", ".7z", ".tar", ".gz"},
    "代码": {".py", ".js", ".ts", ".java", ".html", ".css", ".json", ".xml"}
}

class FileStatsHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        parsed_path = urlparse(self.path)
        
        if parsed_path.path == '/open_folder':
            root = tk.Tk()
            root.withdraw()
            root.attributes('-topmost', True)
            folder_path = filedialog.askdirectory(title="请选择要扫描的工作目录")
            root.destroy()
            self.send_response(200)
            self.send_header('Content-Type', 'application/json; charset=utf-8')
            self.send_header('Access-Control-Allow-Origin', '*')
            self.end_headers()
            self.wfile.write(json.dumps({"path": folder_path}).encode('utf-8'))
            return

        if parsed_path.path == '/scan':
            params = parse_qs(parsed_path.query)
            target_path = params.get('path', [None])[0]
            start_time = float(params.get('start_time', [0])[0])
            end_time = float(params.get('end_time', [float('inf')])[0])
            
            if not target_path or not os.path.isdir(target_path):
                self.send_response(400)
                self.end_headers()
                self.wfile.write(b'Invalid directory path')
                return

            file_count = 0
            dir_count = 0
            ext_stats = defaultdict(list)
            type_stats = defaultdict(list)

            for root, dirs, files in os.walk(target_path):
                dir_count += len(dirs)
                for file in files:
                    file_path = os.path.join(root, file)
                    try:
                        stat = os.stat(file_path)
                        size = stat.st_size
                        mtime = stat.st_mtime
                    except Exception:
                        continue

                    # 时间范围过滤
                    if mtime < start_time or mtime > end_time:
                        continue

                    file_count += 1
                    ext = os.path.splitext(file)[1].lower() or '(无后缀)'
                    file_info = {"path": file_path, "name": file, "size": size, "mtime": mtime}
                    ext_stats[ext].append(file_info)

                    matched_type = False
                    for file_type, extensions in FILE_TYPE_MAP.items():
                        if ext in extensions:
                            type_stats[file_type].append(file_info)
                            matched_type = True
                            break
                    if not matched_type:
                        type_stats["其他"].append(file_info)

            sorted_exts = sorted(ext_stats.items(), key=lambda x: len(x[1]), reverse=True)
            sorted_types = sorted(type_stats.items(), key=lambda x: len(x[1]), reverse=True)
            
            type_result = []
            for t_name, t_files in sorted_types:
                percentage = round((len(t_files) / file_count) * 100, 2) if file_count > 0 else 0
                type_result.append({"type": t_name, "count": len(t_files), "percentage": percentage, "files": t_files})

            result = {
                "total_files": file_count,
                "total_dirs": dir_count,
                "extensions": [{"ext": ext, "count": len(files), "files": files} for ext, files in sorted_exts],
                "file_types": type_result
            }

            self.send_response(200)
            self.send_header('Content-Type', 'application/json; charset=utf-8')
            self.send_header('Access-Control-Allow-Origin', '*')
            self.end_headers()
            self.wfile.write(json.dumps(result, ensure_ascii=False).encode('utf-8'))
        else:
            self.send_response(404)
            self.end_headers()

if __name__ == '__main__':
    server = HTTPServer(('127.0.0.1', 8000), FileStatsHandler)
    print("Server running on http://127.0.0.1:8000")
    server.serve_forever()