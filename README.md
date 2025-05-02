import tkinter as tk
from tkinter import ttk
import serial
import threading
import subprocess
import re
import matplotlib.pyplot as plt
from matplotlib.backends.backend_tkagg import FigureCanvasTkAgg
from matplotlib.animation import FuncAnimation

# Configuration
SERIAL_PORT = 'COM9'  # Set to your USB-to-UART port
BAUD_RATE = 115200
IPERF_SERVER = '192.168.1.4'  # Replace with your Raspberry Pi IP
IPERF_DURATION = 0  # 0 means run continuously

class SerialReader(threading.Thread):
    def __init__(self, callback, port=SERIAL_PORT, baud=BAUD_RATE):
        super().__init__(daemon=True)
        self.callback = callback
        self.ser = serial.Serial(port, baud, timeout=1)

    def run(self):
        while True:
            try:
                line = self.ser.readline().decode(errors='ignore').strip()
                if line.startswith("STATS"):
                    parts = line.split(',')
                    if len(parts) == 3:
                        _, req, grant = parts
                        self.callback(int(req.strip()), int(grant.strip()))
            except Exception as e:
                print(f"Serial error: {e}")

    def send_command_async(self, cmd: str):
        def send():
            try:
                print(f"Sending: {cmd}")
                self.ser.write((cmd + '\n').encode())
            except Exception as e:
                print(f"Write error: {e}")
        threading.Thread(target=send, daemon=True).start()

class IperfPlotter:
    def __init__(self, frame):
        self.times = []
        self.bandwidths = []
        self.time_index = 0

        self.fig, self.ax = plt.subplots(figsize=(5, 3))
        self.canvas = FigureCanvasTkAgg(self.fig, master=frame)
        self.canvas.get_tk_widget().pack(fill=tk.BOTH, expand=True)

        self.ani = FuncAnimation(self.fig, self.update_plot, interval=1000)
        self.run_iperf()

    def run_iperf(self):
        def target():
            cmd = ["iperf3", "-c", IPERF_SERVER, "-i", "1"]
            if IPERF_DURATION > 0:
                cmd += ["-t", str(IPERF_DURATION)]
            process = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.STDOUT, text=True)
            for line in process.stdout:
                match = re.search(r'\[\s*\d+\]\s+\d+\.\d+-\d+\.\d+\s+sec\s+\S+\s+\S+\s+(\d+\.\d+)\s+Mbits/sec', line)
                if match:
                    bandwidth = float(match.group(1))
                    self.times.append(self.time_index)
                    self.bandwidths.append(bandwidth)
                    self.time_index += 1
        threading.Thread(target=target, daemon=True).start()

    def update_plot(self, i):
        self.ax.clear()
        self.ax.plot(self.times, self.bandwidths, marker='o')
        self.ax.set_xlabel('Time (s)')
        self.ax.set_ylabel('Throughput (Mbits/sec)')
        self.ax.set_title('iPerf3 Live Throughput')
        self.ax.grid(True)
        self.canvas.draw()

class StatsGUI:
    def __init__(self, root):
        self.root = root
        self.root.title("ESP32 Grant Control + iPerf3 Plot")

        self.reader = SerialReader(self.update_stats)
        self.reader.start()

        # Left control panel
        left_frame = tk.Frame(root)
        left_frame.pack(side=tk.LEFT, padx=10, pady=10)

        self.req_label = tk.Label(left_frame, text="Requests: 0", font=("Arial", 16))
        self.req_label.pack(pady=5)

        self.grant_label = tk.Label(left_frame, text="Grants: 0", font=("Arial", 16))
        self.grant_label.pack(pady=5)

        tk.Button(left_frame, text="Always Grant",
                  command=lambda: self.reader.send_command_async("CMD,GRANT_MODE,ALWAYS")).pack(fill='x', pady=5)
        tk.Button(left_frame, text="No Grant",
                  command=lambda: self.reader.send_command_async("CMD,GRANT_MODE,NONE")).pack(fill='x', pady=5)
        tk.Button(left_frame, text="Random Grant",
                  command=lambda: self.reader.send_command_async("CMD,GRANT_MODE,RANDOM")).pack(fill='x', pady=5)

        # Right plot panel
        right_frame = tk.Frame(root)
        right_frame.pack(side=tk.LEFT, fill=tk.BOTH, expand=True)
        self.plotter = IperfPlotter(right_frame)

        self.last_req = -1
        self.last_grant = -1

    def update_stats(self, req, grant):
        if req != self.last_req:
            self.req_label.config(text=f"Requests: {req}")
            self.last_req = req
        if grant != self.last_grant:
            self.grant_label.config(text=f"Grants: {grant}")
            self.last_grant = grant

if __name__ == "__main__":
    root = tk.Tk()
    app = StatsGUI(root)
    root.mainloop()
