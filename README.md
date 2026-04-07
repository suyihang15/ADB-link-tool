# ADB-link-tool
用于有电脑上用ADB连接手机的需求的人，做了简单的图形化界面
注:这是本人用adb连接手机用python写的图像界面，使用前请先用adb连接好要控制的手机如果不知道怎么连接的可以看我这篇文章电脑adb连接模拟器，本项目以上传到本人Github仓库，但因为是python的原因可能会有一点延迟。

Github仓库：

打包好的文件下载地址：

1、环境准备

python

2、源码介绍

import subprocess
import cv2
import numpy as np
import tkinter as tk
from PIL import Image, ImageTk
import threading
import time

# 改成你自己的 adb 路径
ADB_PATH = r"\adb.exe" #填你下载的adb.exe文件的地址，不然后面的都不行的，记住
WIN_W = 480
WIN_H = 950

class PhoneControl:
    def __init__(self, root):
        self.root = root
        self.root.title("手机联机软件（作者Mr.苏）")
        self.root.geometry(f"{WIN_W}x{WIN_H}")

        # 1. 投屏显示区域：用于显示手机画面
        self.screen = tk.Label(root)
        self.screen.pack(fill=tk.BOTH, expand=True, padx=10, pady=5)
        self.screen.bind("<Button-1>", self.click) # 绑定鼠标点击事件

        # 2. 功能按键区：返回、主页、唤醒等
        btn_frame = tk.Frame(root)
        btn_frame.pack(pady=5)
        tk.Button(btn_frame, text="返回", command=self.back).grid(row=0, column=0, padx=3)
        tk.Button(btn_frame, text="主页", command=self.home).grid(row=0, column=1, padx=3)
        tk.Button(btn_frame, text="唤醒", command=self.wake).grid(row=0, column=2, padx=3)
        tk.Button(btn_frame, text="电源", command=self.power).grid(row=0, column=3, padx=3)
        tk.Button(btn_frame, text="截图", command=self.screenshot).grid(row=0, column=4, padx=3)

        # 3. ADB 命令终端区：输入自定义命令
        term_frame = tk.Frame(root)
        term_frame.pack(fill=tk.X, padx=10, pady=5)
        tk.Label(term_frame, text="ADB命令：").pack(side=tk.LEFT)
        self.cmd_input = tk.Entry(term_frame)
        self.cmd_input.pack(side=tk.LEFT, fill=tk.X, expand=True, padx=5)
        tk.Button(term_frame, text="执行", command=self.run_cmd).pack(side=tk.LEFT)

        # 命令输出框：显示命令执行结果
        self.output_box = tk.Text(root, height=6)
        self.output_box.pack(fill=tk.BOTH, padx=10, pady=5)

        self.running = True
        self.current_frame = None

        # 启动投屏线程
        threading.Thread(target=self.stream, daemon=True).start()
        # 启动声音转发线程
        threading.Thread(target=self.audio, daemon=True).start()
        # 启动界面刷新循环
        self.update_ui()

    # ---------------------- 核心功能模块 ----------------------
    # 投屏流获取：不断从手机抓取屏幕数据（核心循环）
    def stream(self):
        while self.running:
            try:
                # 使用 ADB  exec-out 命令抓取手机屏幕二进制流
                data = subprocess.run([ADB_PATH, "exec-out", "screencap", "-p"], stdout=subprocess.PIPE).stdout
                # 数据过短表示抓取失败，跳过
                if len(data) < 1000:
                    continue
                # 将二进制流转为 numpy 数组
                nparr = np.frombuffer(data, np.uint8)
                # 解码为图片
                img = cv2.imdecode(nparr, cv2.IMREAD_COLOR)
                # 转换颜色格式 (BGR -> RGB) 并赋值给当前帧变量
                if img is not None:
                    self.current_frame = cv2.cvtColor(img, cv2.COLOR_BGR2RGB)
            except:
                pass
            # 控制抓取频率，0.008秒 = 125帧左右
            time.sleep(0.008)

    # UI 画面刷新：将 current_frame 显示在 Label 上
    def update_ui(self):
        if self.current_frame is not None:
            try:
                h, w = self.current_frame.shape[:2]
                # 计算缩放比例，保证画面适应窗口且不变形
                scale = min(WIN_W/w, 850/h)
                img = cv2.resize(self.current_frame, (int(w*scale), int(h*scale)))
                # 转为 Tkinter 可用的图片格式
                imgtk = ImageTk.PhotoImage(Image.fromarray(img))
                # 更新界面显示
                self.screen.config(image=imgtk)
                self.screen.imgtk = imgtk # 强制保存引用，防止被垃圾回收
            except:
                pass
        # 递归调用，实现每 8 毫秒刷新一次界面
        if self.running:
            self.root.after(8, self.update_ui)

    # 鼠标点击控制：将电脑鼠标坐标映射为手机坐标并发送点击指令
    def click(self, e):
        if self.current_frame is None:
            return
        h, w = self.current_frame.shape[:2]
        # 计算真实的手机点击位置
        x = int(e.x * w / WIN_W)
        y = int(e.y * h / 850)
        # 执行 ADB 点击命令
        subprocess.Popen([ADB_PATH, "shell", "input", "tap", str(x), str(y)])

    # 音频转发：通过 screenrecord 转发手机声音流
    def audio(self):
        while self.running:
            try:
                # 启动手机录屏并输出到标准输出（用于转发声音）
                subprocess.Popen([ADB_PATH, "shell", "screenrecord", "--output-format", "h264", "-"],
                                stdout=subprocess.PIPE, stderr=subprocess.DEVNULL)
            except:
                pass
            time.sleep(1)

    # ---------------------- 基础按键功能模块 ----------------------
    # 模拟手机返回键
    def back(self):
        subprocess.Popen([ADB_PATH, "shell", "input", "keyevent", "4"])
    # 模拟手机主页键
    def home(self):
        subprocess.Popen([ADB_PATH, "shell", "input", "keyevent", "3"])
    # 模拟多任务键
    def recent(self):
        subprocess.Popen([ADB_PATH, "shell", "input", "keyevent", "187"])
    # 模拟唤醒屏幕/点亮屏幕
    def wake(self):
        subprocess.Popen([ADB_PATH, "shell", "input", "keyevent", "224"])
    # 模拟电源键（锁屏/点亮）
    def power(self):
        subprocess.Popen([ADB_PATH, "shell", "input", "keyevent", "26"])

    # 截图功能：截取手机屏幕并保存到电脑
    def screenshot(self):
        try:
            # 先在手机内存截图
            subprocess.run([ADB_PATH, "shell", "screencap", "/sdcard/screenshot.png"])
            # 拉取到电脑当前目录
            subprocess.run([ADB_PATH, "pull", "/sdcard/screenshot.png", "screenshot.png"])
            self.output_box.insert(tk.END, "\n截图已保存到当前目录：screenshot.png")
        except:
            self.output_box.insert(tk.END, "\n截图失败")

    # ADB 命令执行：读取输入框的命令并执行
    def run_cmd(self):
        cmd = self.cmd_input.get().strip()
        if not cmd:
            return
        try:
            # 拆分命令并执行
            res = subprocess.run([ADB_PATH] + cmd.split(), stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True)
            # 输出结果
            self.output_box.insert(tk.END, f"\n> {cmd}\n{res.stdout}{res.stderr}")
            self.output_box.see(tk.END) # 滚动到最底部
        except Exception as e:
            self.output_box.insert(tk.END, f"\n错误：{str(e)}")

if __name__ == "__main__":
    root = tk.Tk()
    app = PhoneControl(root)
    root.mainloop()
3、运行

（1）安装所需第三方库 pip install opencv-python numpy pillow

（2）运行python文件

4、效果展示


这些就是全部功能，如果想直接使用就直接用我打包好的exe文件就行。
