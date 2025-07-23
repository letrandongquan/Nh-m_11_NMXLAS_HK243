## Object capture ( phím space )
## Mục Đích
- Cho phép người dùng lưu lại hình ảnh đang hiển thị từ camera bằng cách nhấn phím Space (phím cách).
- Nếu đang ở chế độ đo đạc, ảnh sẽ được ghi lại kèm theo thông tin đo chiều dài/rộng và thời gian.
- Giúp lưu trữ dữ liệu dưới dạng ảnh để phân tích, báo cáo hoặc kiểm tra sau.
## Ý tưởng thực hiện 
- khi người dùng nhấn phím cách, chương trình sẽ hoạt động như sau:
1. ghi lại thời điểm chụp lúc đó để làm tên file
2. lưu ảnh gốc ( frame).
3. nếu đang sử dụng các tính năng vào vật thể, lấy luôn các khuôn, số liệu, và các thông tin vật thể vào ảnh
4. ghi ảnh ra ổ đĩa với định dạng và chất lượng đã thiết lập
# Công thức
# công thức xử lí đo đạc
- (x1, y1): điểm bắt đầu khi rê chuột.
- (x2, y2): điểm hiện tại con trỏ chuột.
- cx, cy: tọa độ tâm ảnh (để quy đổi về hệ tọa độ chuẩn).
- conv(): hàm chuyển đổi tọa độ pixel sang đơn vị đo thực tế (ví dụ: mm hoặc cm).
- Quy đổi:
```python
x1' = x1 + cx
x2' = x2 + cx
y1' = -y1 + cy
y2' = -y2 + cy
```
# ví dụ hoạt động 
1. Khi người dùng đo một vật thể có chiều ngang và dọc, rồi nhấn phím cách (space)
2. Một ảnh sẽ được lưu vào folder capture_objects, tên của ảnh giống như thế này vd: object_20250722_002131.jpg
3. trong ảnh có thể chỉ là hình chụp vật thể hoặc đi kèm với các tính năng đã sử dụng như khung chữ nhật đánh dấu vật thể, số liệu đo, chiều dài tính theo đơn vị,...
# code chính
```python
elif key == 32:  # space bar
    # Tạo tên file với timestamp
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    filename = os.path.join(capture_folder, f"object_{timestamp}.{capture_format}")
    
    # Tạo thư mục nếu chưa có
    if not os.path.exists(capture_folder):
        os.makedirs(capture_folder)
    
    # Sao chép frame hiện tại
    frame_copy = frame0.copy()
    
    # Nếu đang đo và không ở chế độ tự động hay đếm
    if mouse_mark and not key_flags['auto'] and not key_flags['count']:
        x1, y1 = mouse_mark
        x2, y2 = mouse_now
        
        # Quy đổi tọa độ chuột sang tọa độ thực tế trên ảnh
        x1 += cx; x2 += cx
        y1 = -y1 + cy
        y2 = -y2 + cy
        
        # Vẽ hình chữ nhật đo đạc và đường nối
        cv2.rectangle(frame_copy, (x1, y1), (x2, y2), (0, 0, 255), 2)
        cv2.line(frame_copy, (x1, y1), (x2, y2), (0, 255, 0), 2)
        
        # Tính chiều dài X và Y sau khi chuyển đổi đơn vị
        x1c, y1c = conv(x1 - cx, (y1 - cy) * -1)
        x2c, y2c = conv(x2 - cx, (y2 - cy) * -1)
        xlen = abs(x1c - x2c)
        ylen = abs(y1c - y2c)
        
        # Hiển thị thông tin lên ảnh
        cv2.putText(frame_copy, f"X: {xlen:.2f}{unit_suffix}", (10, 30), 
                    cv2.FONT_HERSHEY_SIMPLEX, 0.7, (0, 0, 255), 2)
        cv2.putText(frame_copy, f"Y: {ylen:.2f}{unit_suffix}", (10, 60), 
                    cv2.FONT_HERSHEY_SIMPLEX, 0.7, (0, 0, 255), 2)
        cv2.putText(frame_copy, f"Time: {timestamp}", (10, 90), 
                    cv2.FONT_HERSHEY_SIMPLEX, 0.5, (255, 255, 255), 1)

    # Ghi ảnh ra file theo định dạng và chất lượng đã chọn
    if capture_format.lower() in ('jpg', 'jpeg'):
        cv2.imwrite(filename, frame_copy, [int(cv2.IMWRITE_JPEG_QUALITY), capture_quality])
    else:
        cv2.imwrite(filename, frame_copy)
    
    print(f"CAPTURE: Saved object image to {filename}")
```

## chức năng nhận diện hình học - shape detection ( phím s )
# mục đích
- tự động phân loại các hình cơ bản như: tam giác, hình vuông, chữ nhật, hình tròn,...
- hữu ích trong việc ứng dụng đo lường, phân loại vật thể, nhận dạng mẫu,...
# ý tưởng thực hiện
- Dùng xử lí ảnh để tìm biên đối tượng
- Dựa vào số lượng đỉnh (vertices) và tỷ lệ cạnh, xác định hình dạng.
- Đối Với hình tròn, kiểm tra độ tròn (circularity) dựa vào diện tích và chu vi.
# Công thức toán học đã xử dụng
- Chu vi (Perimeter) của đường bao:
```python
P = cv2.arcLength(contour, True)
```
- Gần đúng đa giác (Polygon approximation):
```python
approx = cv2.approxPolyDP(contour, 0.04 * P, True)
```
- Tỷ lệ khung chữ nhật (Aspect Ratio) – phân biệt vuông và chữ nhật:
```python
aspect_ratio = width / height
```
- Độ tròn (Circularity) – dùng để phân biệt hình tròn:
```python
circularity = 4 * π * area / (perimeter)^2 
```
- nếu tính ra > 0.8 thì là hình tròn, nhỏ hơn thì là elip hoặc hình khác
# ví dụ
- vật có 3 đỉnh gắn label là triangle, 4 thì là square, tròn là circle,...
# code chính
```python 
def classify_shapes(frame):
    # Chuyển ảnh sang xám và làm mờ nhẹ
    gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
    blurred = cv2.GaussianBlur(gray, (5, 5), 0)
    
    # Nhị phân ảnh và đảo ngược màu
    _, thresh = cv2.threshold(blurred, 127, 255, cv2.THRESH_BINARY)
    thresh = ~thresh  # đảo màu trắng đen
    
    # Tìm đường bao (contours)
    contours, _ = cv2.findContours(thresh, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
    
    for c in contours:
        # Bỏ qua các vùng quá nhỏ
        x, y, w, h = cv2.boundingRect(c)
        if w * h < 100:
            continue

        # Gần đúng đa giác từ contour
        peri = cv2.arcLength(c, True)
        approx = cv2.approxPolyDP(c, 0.04 * peri, True)
        
        # Tâm đối tượng
        cx = x + w // 2
        cy = y + h // 2
        
        # Phân loại hình dựa theo số đỉnh
        shape = "unknown"
        if len(approx) == 3:
            shape = "triangle"
        elif len(approx) == 4:
            aspect_ratio = float(w) / h
            shape = "square" if 0.95 <= aspect_ratio <= 1.05 else "rectangle"
        elif len(approx) == 5:
            shape = "pentagon"
        elif len(approx) == 6:
            shape = "hexagon"
        else:
            area = cv2.contourArea(c)
            circularity = 4 * np.pi * area / (peri * peri)
            shape = "circle" if circularity > 0.8 else "oval"
        
        # Vẽ khung và tên hình dạng
        draw.rect(frame, x, y, x+w, y+h, weight=1, color='blue')
        draw.add_text(frame, shape, cx, cy, color='yellow', center=True)

    return frame

```
