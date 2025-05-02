import subprocess
import threading
import re
import tkinter as tk
from matplotlib.backends.backend_tkagg import FigureCanvasTkAgg
import matplotlib.pyplot as plt
from matplotlib.animation import FuncAnimation

class Iperf3Plotter:
    def __init__(self, root):
        self.root = root
        self.root.title("iPerf3 Real-Time Bandwidth Monitor")

        self.bandwidths = []
        self.time_points = []

        # Create plot
        self.fig, self.ax = plt.subplots(figsize=(6, 4))
        self.ax.set_title("iPerf3 Bandwidth (Mbits/sec)")
        self.ax.set_xlabel("Seconds")
        self.ax.set_ylabel("Bandwidth")

        # Embed plot into Tkinter window
        self.canvas = FigureCanvasTkAgg(self.fig, master=root)
        self.canvas.get_tk_widget().pack(side=tk.RIGHT, fill=tk.BOTH, expand=True)

        # Start the iperf3 process in background
        threading.Thread(target=self.run_iperf3, daemon=True).start()

        # Animate the plot
        self.ani = FuncAnimation(self.fig, self.update_plot, interval=1000)

    def run_iperf3(self):
        cmd = [
            "C:\\Users\\rk52524\\Downloads\\iperf3.12_64\\iperf3.exe",
            "-c", "192.168.1.4",  # CHANGE to your server IP
            "-p", "5202",
            "-i", "1",
            "-t", "9999"
        ]
        try:
            process = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.STDOUT, text=True)
            for line in iter(process.stdout.readline, ''):
                print(f"[iperf3 stdout] {line.strip()}")
                self.parse_output(line.strip())
        except FileNotFoundError:
            print("ERROR: iperf3 executable not found. Check the path.")
        except Exception as e:
            print(f"Error running iperf3: {e}")

    def parse_output(self, line):
        match = re.search(r'\[\s*\d+\]\s+\d+\.\d+-\d+\.\d+\s+sec\s+\d+\s+\w+Bytes\s+([\d.]+)\s+Mbits/sec', line)
        if match:
            bandwidth = float(match.group(1))
            self.bandwidths.append(bandwidth)
            self.time_points.append(len(self.bandwidths))

    def update_plot(self, frame):
        print("update_plot() called")
        print(f"Data points: {len(self.bandwidths)}")
        if self.bandwidths:
            print(f"Last value: {self.bandwidths[-1]}")
        else:
            print("Last value: None")

        self.ax.clear()
        self.ax.set_title("iPerf3 Bandwidth (Mbits/sec)")
        self.ax.set_xlabel("Seconds")
        self.ax.set_ylabel("Bandwidth")
        self.ax.plot(self.time_points, self.bandwidths, marker='o')
        self.ax.grid(True)

if __name__ == "__main__":
    root = tk.Tk()
    app = Iperf3Plotter(root)
    root.mainloop()
