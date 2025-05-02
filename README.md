import tkinter as tk
from tkinter import ttk
import serial
import threading
import subprocess
import re
import matplotlib.pyplot as plt
from matplotlib.backends.backend_tkagg import FigureCanvasTkAgg
from matplotlib.animation import FuncAnimation

# === CONFIGURATION ===
SERIAL_PORT = 'COM9'
BAUD_RATE = 115200
IPERF_PATH = 'C:\\Users\\rk52524\\Downloads\\iperf3.12_64\\iperf3.exe'
IPERF_SERVER = '192.168.1.4'
IPERF_PORT = 5202

# === SERIAL THREAD ===
class SerialReader(threading.Thread):
    def __init__(self, callback):
        super().__init__(daemon=True)
        self.callback = callback
        self.ser = serial.Serial(SERIAL_PORT, BAUD_RATE, timeout=1)

    def run(self):
        while True:
            try:
                line = self.ser.readline().decode(errors='ignore').strip()
                if line.startswith("STATS"):
                    parts = line.split(',')
                    if len(parts) == 3:
                        _, req, grant = parts
                        self.callback(int(req), int(grant))
            except Exception as e:
                print("Serial error:", e)

    def send_command(self, command):
        try:
            print("Sending:", command)
            self.ser.write((command + '\n').encode())
        except Exception as e:
            print("Send error:", e)

# === GUI ===
class GUIApp:
    def __init__(self, root):
        self.root = root
        self.root.title("ESP32 Grant GUI + iperf3 Monitor")

        self.reader = SerialReader(self.update_stats)
        self.reader.start()

        self.req_count = 0
        self.grant_count = 0

        self.bandwidth = []
        self.time_points = []

        self.setup_widgets()
        self.start_iperf_thread()
        self.setup_plot()

    def setup_widgets(self):
        top = tk.Frame(self.root)
        top.pack(side=tk.LEFT, padx=10, pady=10)

        self.req_label = tk.Label(top, text="Requests: 0", font=("Arial", 14))
        self.req_label.pack(pady=5)

        self.grant_label = tk.Label(top, text="Grants: 0", font=("Arial", 14))
        self.grant_label.pack(pady=5)

        tk.Button(top, text="Always Grant", command=lambda: self.set_grant_mode("ALWAYS")).pack(pady=2)
        tk.Button(top, text="No Grant", command=lambda: self.set_grant_mode("NONE")).pack(pady=2)
        tk.Button(top, text="Random Grant", command=lambda: self.set_grant_mode("RANDOM")).pack(pady=2)

    def set_grant_mode(self, mode):
        self.reader.send_command(f"CMD,GRANT_MODE,{mode}")

    def update_stats(self, req, grant):
        self.req_count = req
        self.grant_count = grant
        self.req_label.config(text=f"Requests: {req}")
        self.grant_label.config(text=f"Grants: {grant}")

    def start_iperf_thread(self):
        def target():
            cmd = [IPERF_PATH, '-c', IPERF_SERVER, '-p', str(IPERF_PORT), '-i', '1', '-t', '9999']
            process = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.STDOUT, text=True)
            for line in process.stdout:
                rate = self.extract_bandwidth(line)
                if rate is not None:
                    self.bandwidth.append(rate)
                    self.time_points.append(len(self.time_points))

        threading.Thread(target=target, daemon=True).start()

    def extract_bandwidth(self, line):
        match = re.search(r'\s+(\d+\.\d+)\s+Mbits/sec', line)
        if match:
            return float(match.group(1))
        return None

    def setup_plot(self):
        self.fig, self.ax = plt.subplots(figsize=(5, 3))
        self.ax.set_title("iperf3 Bandwidth (Mbit/s)")
        self.ax.set_xlabel("Time (s)")
        self.ax.set_ylabel("Throughput")
        self.canvas = FigureCanvasTkAgg(self.fig, master=self.root)
        self.canvas.get_tk_widget().pack(side=tk.RIGHT, padx=10)
        self.ani = FuncAnimation(self.fig, self.update_plot, interval=1000)

    def update_plot(self, frame):
        self.ax.clear()
        self.ax.set_title("iperf3 Bandwidth (Mbit/s)")
        self.ax.set_xlabel("Time (s)")
        self.ax.set_ylabel("Throughput")
        self.ax.plot(self.time_points, self.bandwidth, color='blue')
        self.ax.grid(True)

# === MAIN ===
if __name__ == "__main__":
    root = tk.Tk()
    app = GUIApp(root)
    root.mainloop()
