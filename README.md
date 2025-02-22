# Neuro-City
AI-Driven traffic management system using Azure services
Traffic Management System with Azure Integration

This project is a *real-time traffic management system* that leverages Azure services for vehicle detection, data storage, and IoT communication. It uses Azure Computer Vision, Data Lake Storage, and IoT Hub to analyze traffic density, store data, and control traffic signals dynamically.

## Features
- Real-time vehicle detection using Azure Computer Vision
- Data storage in Azure Data Lake Storage Gen2
- IoT Hub integration for telemetry data
- Dynamic traffic signal control based on vehicle count
- Error handling and logging for robust operation

## Prerequisites
- Azure account with required services (Computer Vision, Data Lake, IoT Hub)
- Python 3.8+
- OpenCV for video capture
**main.py**
import cv2
import time
import json
import pandas as pd
from utils.azure_utils import analyze_traffic
from utils.iot_utils import send_iot_message
from utils.data_lake_utils import upload_to_data_lake

def control_traffic(vehicle_count):
    if vehicle_count < 10:
        signal_time = 30  # Short wait for minimal traffic
    elif vehicle_count < 30:
        signal_time = 60  # Moderate wait time
    else:
        signal_time = 120  # Maximum wait time to avoid congestion
    return signal_time

def main():
    camera = cv2.VideoCapture(0)
    while True:
        ret, frame = camera.read()
        if not ret:
            break

        image_path = "traffic_image.jpg"
        cv2.imwrite(image_path, frame)

        vehicle_count = analyze_traffic(image_path)
        signal_time = control_traffic(vehicle_count)

        # Upload data to Data Lake
        data = {
            "timestamp": time.strftime("%Y-%m-%d %H:%M:%S"),
            "vehicle_count": vehicle_count,
            "signal_time": signal_time
        }
        upload_to_data_lake(data)

        # Send data to IoT Hub
        send_iot_message(vehicle_count)

        print(f"Vehicle Count: {vehicle_count}, Signal Time: {signal_time} seconds")
        time.sleep(signal_time)

    camera.release()
    cv2.destroyAllWindows()

if _name_ == "_main_":
    main()
    
**utils/azure_utils.py**
import requests
from config import AZURE_VISION_ENDPOINT, AZURE_VISION_KEY

def analyze_traffic(image_path):
    headers = {
        'Ocp-Apim-Subscription-Key': AZURE_VISION_KEY,
        'Content-Type': 'application/octet-stream'
    }
    with open(image_path, 'rb') as image:
        response = requests.post(
            f"{AZURE_VISION_ENDPOINT}/vision/v3.1/analyze?visualFeatures=Objects",
            headers=headers, data=image
        )
    analysis = response.json()
vehicle_count = sum(1 for obj in analysis.get('objects', []) if obj['object'] in ['car', 'truck', 'bus'])
    return vehicle_count
    
**utils/data_lake_utils.py**
from azure.storage.filedatalake import DataLakeServiceClient
from config import DATA_LAKE_ACCOUNT_NAME, DATA_LAKE_KEY, DATA_LAKE_CONTAINER_NAME
import pandas as pd
import time

def upload_to_data_lake(data):
    service_client = DataLakeServiceClient(
        account_url=f"https://{DATA_LAKE_ACCOUNT_NAME}.dfs.core.windows.net",
        credential=DATA_LAKE_KEY
    )
    file_system_client = service_client.get_file_system_client(DATA_LAKE_CONTAINER_NAME)
    date_partition = time.strftime("%Y/%m/%d")
    file_path = f"{date_partition}/traffic_data_{int(time.time())}.csv"

    df = pd.DataFrame([data])
    file_client = file_system_client.create_file(file_path)
    file_client.upload_data(df.to_csv(index=False), overwrite=True)
    
**utils/iot_utils.py**
from azure.iot.device import IoTHubDeviceClient, Message
from config import IOT_HUB_CONNECTION_STRING
def send_iot_message(vehicle_count):
    try:
        iot_client = IoTHubDeviceClient.create_from_connection_string(IOT_HUB_CONNECTION_STRING)
        message = Message(json.dumps({"vehicle_count": vehicle_count}))
        iot_client.send_message(message)
        print("Message sent to IoT Hub")
    except Exception as e:
        print(f"Error sending IoT message: {e}")
**docs/azure_resource_setup.md**
markdown
