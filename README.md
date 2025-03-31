import matplotlib.pyplot as plt

timestamps = []
bitrate = []

with open("iperf3_output.txt", "r") as file:
    for line in file:
        if "sec" in line and "Mbits/sec" in line:
            try:
                # Example line format:
                # [  5]  0.00-0.10 sec  1.32 MBytes  111 Mbits/sec  0  KBytes
                parts = line.split()
                time_range = parts[2]
                mbps = float(parts[6])  # Bitrate in Mbits/sec

                # Extract mid-point of the interval (e.g., 0.05 for 0.00-0.10)
                t_start, t_end = map(float, time_range.split('-'))
                mid_time = (t_start + t_end) / 2

                timestamps.append(mid_time)
                bitrate.append(mbps)
            except (IndexError, ValueError):
                continue  # Skip malformed lines

# Plotting
plt.figure(figsize=(10, 5))
plt.plot(timestamps, bitrate, marker='o')
plt.xlabel("Time (s)")
plt.ylabel("Throughput (Mbits/sec)")
plt.title("WiFi Throughput Over Time")
plt.grid(True)
plt.tight_layout()
plt.show()
