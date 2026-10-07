BÀI THỰC HÀNH SỐ 2 – THU THẬP, LƯU TRỮ VÀ TIỀN XỬ LÝ DỮ LIỆU IoT
1. Giới thiệu
Project xây dựng một pipeline IoT hoàn chỉnh từ thiết bị cảm biến đến dashboard giám sát:
DHT22 + HC-SR04
       ↓
      ESP32
       ↓ MQTT/JSON
 broker.emqx.io
       ↓
Python Collector
       ↓
   InfluxDB 2.x
   ├── sensor_raw
   └── sensor_processed
       ↓
Streamlit Dashboard
Hệ thống thực hiện:
- Thu thập nhiệt độ, độ ẩm và khoảng cách từ ESP32.
- Publish dữ liệu JSON qua MQTT.
- Python subscribe dữ liệu, validate, kiểm tra sequence và ghi vào InfluxDB.
- Tiền xử lý dữ liệu: missing values, outlier IQR, resampling, rolling mean, delta, Min-Max normalization.
- Lưu dữ liệu đã xử lý vào measurement mới.
- Hiển thị dữ liệu real-time và dữ liệu đã xử lý bằng Streamlit.
2. Phần cứng mô phỏng
Project sử dụng Wokwi và ESP32 DevKit V1.
DHT22
Chân	ESP32
VCC	3V3
GND	GND
DATA	GPIO 15


Có điện trở pull-up 10 kΩ giữa DATA và 3V3.
HC-SR04
Chân	ESP32
VCC	VIN
GND	GND
TRIG	GPIO 5
ECHO	GPIO 18


LED
GPIO 2 → điện trở 220 Ω → LED → GND
LED được dùng để báo trạng thái publish MQTT thành công.
3. MQTT
Broker
broker.emqx.io
Port
1883
Topic
ptit/iot/B23DCAT289/esp32/sensor
JSON mẫu
{
  "device_id": "ESP32-B23DCAT289",
  "temperature": 25.0,
  "humidity": 50.0,
  "distance_cm": 100.82,
  "rssi": -85,
  "sequence": 1,
  "uptime_s": 10,
  "ts_ms": 1790000000000
}
4. Cấu trúc thư mục
IoT_BaiTH2/
│
├── src/
│   └── sketch.ino
│
├── python/
│   ├── config.py
│   ├── mqtt_to_influx.py
│   ├── preprocess.py
│   ├── dashboard.py
│   └── test_influx.py
│
├── output/
│   ├── raw_data.csv
│   └── processed_data.csv
│
├── include/
├── lib/
├── test/
│
├── diagram.json
├── platformio.ini
├── wokwi.toml
│
├── .env
├── .env.example
├── .gitignore
├── requirements.txt
└── README.md
.env, venv/ và các thông tin bí mật không được đẩy lên GitHub.

5. Cơ sở dữ liệu InfluxDB
InfluxDB chạy tại:
http://localhost:8086
Cấu hình project:
Organization: PTIT
Bucket: iot_data
Measurement sensor_raw
Tag:
device_id
Fields chính:
temperature
humidity
distance_cm
rssi
sequence
uptime_s
receive_ts_ms
latency_ms
Measurement sensor_processed
Fields chính:
temperature
humidity
distance_cm

temperature_rolling_mean
humidity_rolling_mean
distance_rolling_mean

temperature_delta
humidity_delta
distance_delta

temperature_norm
humidity_norm
distance_norm
Retention policy được cấu hình trực tiếp trên bucket iot_data trong InfluxDB theo thời gian lưu trữ mong muốn của bài thực hành.
6. Cài đặt Python
Tạo môi trường ảo
python -m venv venv
Kích hoạt trên Windows PowerShell
.\venv\Scripts\Activate.ps1
Cài thư viện
pip install -r requirements.txt
Các thư viện chính:
- paho-mqtt
- influxdb-client
- pandas
- numpy
- scikit-learn
- streamlit
- streamlit-autorefresh
- python-dotenv
7. Cấu hình .env
Tạo file .env ở thư mục gốc:
MQTT_BROKER=broker.emqx.io
MQTT_PORT=1883
MQTT_TOPIC=ptit/iot/B23DCAT289/esp32/sensor

DEVICE_ID=ESP32-B23DCAT289

INFLUX_URL=http://localhost:8086
INFLUX_TOKEN=YOUR_INFLUXDB_TOKEN
INFLUX_ORG=PTIT
INFLUX_BUCKET=iot_data
Không commit file .env lên GitHub.
8. Cách chạy hệ thống
Bước 1 – Khởi động InfluxDB
Chạy influxd.exe, sau đó kiểm tra:
http://localhost:8086
Bước 2 – Build firmware ESP32
pio run
Sau đó chạy Wokwi Simulator.
ESP32 sẽ:
1. Kết nối WiFi.
2. Kết nối MQTT Broker.
3. Đọc DHT22 và HC-SR04.
4. Tạo JSON.
5. Publish dữ liệu theo chu kỳ 5 giây.
Bước 3 – Chạy MQTT Collector
python python\mqtt_to_influx.py
Collector thực hiện:
MQTT subscribe
    ↓
Decode JSON
    ↓
Validate dữ liệu
    ↓
Kiểm tra duplicate / packet loss
    ↓
Ghi sensor_raw vào InfluxDB
Khi thành công:
[SUCCESS] Saved to InfluxDB
Bước 4 – Tiền xử lý dữ liệu
python python\preprocess.py
Các bước tiền xử lý:
1. Đọc sensor_raw từ InfluxDB.
2. Resampling theo cửa sổ 30 giây.
3. Xử lý missing values bằng interpolation.
4. Phát hiện outlier bằng IQR.
5. Tạo rolling mean.
6. Tính delta.
7. Chuẩn hóa Min-Max.
8. Ghi kết quả vào sensor_processed.
File kết quả:
output/raw_data.csv
output/processed_data.csv
Bước 5 – Chạy dashboard
python -m streamlit run python\dashboard.py
Truy cập:
http://localhost:8501
Dashboard tự động cập nhật theo chu kỳ và hiển thị:
- Nhiệt độ
- Độ ẩm
- Khoảng cách
- RSSI
- Sequence
- Uptime
- E2E latency (khi timestamp hợp lệ)
- Biểu đồ dữ liệu raw
- Rolling mean
- Delta
- Min-Max normalized data
- Bảng các bản ghi mới nhất
9. Tiền xử lý dữ liệu
Missing values
Sau resampling, các khoảng dữ liệu thiếu được xử lý bằng:
Time interpolation
→ Forward fill
→ Backward fill
Outlier bằng IQR
IQR = Q3 - Q1
Lower = Q1 - 1.5 × IQR
Upper = Q3 + 1.5 × IQR
Giá trị ngoài khoảng [Lower, Upper] được xem là outlier, chuyển thành giá trị thiếu và được nội suy lại.
Rolling Mean
Rolling mean sử dụng cửa sổ 3 mẫu để làm mượt dữ liệu.
Delta
delta = current_value - previous_value
Min-Max Normalization
Các trường temperature, humidity, distance_cm được chuẩn hóa về khoảng [0, 1].
10. Kiểm tra tính toàn vẹn dữ liệu
Trường sequence được sử dụng để theo dõi dữ liệu:
Duplicate
Ví dụ:
10 → 11 → 11 → 12
Sequence 11 xuất hiện hai lần.
Packet loss
Ví dụ:
10 → 11 → 13
Có khả năng mất bản tin sequence 12.
Collector có xử lý trường hợp thiết bị restart thông qua sequence và uptime_s.
11. Bộ dữ liệu thử nghiệm
Có thể thay đổi thông số cảm biến Wokwi theo các bộ thử nghiệm sau:
Bộ	Temperature	Humidity	Distance
1	25 °C	50 %	20 cm
2	28 °C	60 %	50 cm
3	32 °C	70 %	100 cm
4	35 °C	80 %	150 cm
5	22 °C	40 %	200 cm


Ngoài ra có thể tạo giá trị bất thường để kiểm tra outlier hoặc tạm dừng thiết bị để kiểm tra missing data.
12. Kết quả
Pipeline hoàn chỉnh:
ESP32/Wokwi
     ↓
MQTT Broker
     ↓
Python Collector
     ↓
InfluxDB
├── sensor_raw
└── sensor_processed
     ↓
Streamlit Dashboard
Hệ thống đáp ứng các chức năng chính:
- Thu thập dữ liệu cảm biến real-time.
- Publish/subscribe qua MQTT.
- Validate dữ liệu.
- Lưu dữ liệu time-series.
- Kiểm tra duplicate và packet loss.
- Tiền xử lý dữ liệu.
- Chuẩn hóa và tạo đặc trưng.
- Lưu dữ liệu đã xử lý.
- Giám sát bằng dashboard Streamlit.
13. Công nghệ sử dụng
- ESP32
- Wokwi
- PlatformIO
- MQTT
- EMQX Public Broker
- Python 3
- Paho MQTT
- InfluxDB 2.x
- Pandas
- NumPy
- Scikit-learn
- Streamlit