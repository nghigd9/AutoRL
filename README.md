# AutoRL

Đây là một PoC (Proof of Concept) nhằm tự động chặn lưu lượng truy cập trên Cloudflare bằng cách phân tích log của Nginx.

Công cụ này sẽ đánh giá tệp `access.log` của Nginx để tìm các mẫu tấn công CC (Challenge Collapsar) tiềm ẩn, sau đó chặn chúng trên Cloudflare và gửi thông báo đến nhóm Telegram.

## Mô hình hoạt động

Với **Cloudflare Argo Tunnel**, chúng ta có thể thiết lập nhóm bảo mật để chỉ cho phép lưu lượng SSH vào hệ thống, đảm bảo rằng IP của máy chủ không bị lộ trên Internet (tham khảo: [Sử dụng Cloudflare Argo Tunnel (cloudflared) để tăng tốc và bảo vệ website của bạn](https://nova.moe/accelerate-and-secure-with-cloudflared/)). Tuy nhiên, kẻ tấn công vẫn có thể thực hiện CC website bằng cách gửi một lượng lớn request đồng thời. **AutoRL** giúp giảm thiểu vấn đề này.

![AutoRL](./AutoRL.png)

## Yêu cầu

Vì đây chỉ là một PoC, nên cần đáp ứng các điều kiện sau để sử dụng **AutoRL**:

- Máy chủ phải cài đặt **Python 3**.
- Máy chủ sử dụng **Nginx** làm reverse proxy, và toàn bộ log được ghi vào một tệp `access.log`.
- **Nginx** phải có định dạng log như sau (trong tệp `/etc/nginx/nginx.conf`):

    ```nginx
    log_format  main  '$remote_addr $time_iso8601 "$request" $server_name '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    ```
    Khi đó, log thô sẽ có dạng:

    ```
    108.162.245.152 2022-05-05T10:14:19+08:00 "GET /grafana/api/live/ws HTTP/1.1" xxxx.yyyy.tld 400 12 "-" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, như Gecko)" "145.xx.xxx.xxx"
    ```

    Trong đó:

    - `108.162.245.152` là IP của Cloudflare.
    - `xxxx.yyyy.tld` là tên miền được yêu cầu.
    - `2022-05-02T10:44:16+08:00` là thời gian yêu cầu.
    - `"145.xx.xx.xxx"` là IP thực của khách truy cập.

## Cách sử dụng

1. **Tải tệp** `autorl.py` về máy chủ.
2. **Chỉnh sửa các biến cấu hình** trong `autorl.py`:

    - `CF_EMAIL`: Email đăng nhập Cloudflare của bạn.
    - `CF_AUTH_KEY`: Global API Key của Cloudflare.
    - `ACCESS_LOG_PATH`: Đường dẫn đến tệp log của Nginx (mặc định là `/var/log/nginx/access.log`).
    - `INTERVAL_MIN`: Khoảng thời gian đánh giá log (mặc định là **1 phút**).
    - `RATE_PER_MINUTE`: Giới hạn số request tối đa mà một IP có thể gửi trong `INTERVAL_MIN` phút (ví dụ: nếu đặt là **600** và `INTERVAL_MIN` là **1**, thì một IP chỉ được gửi tối đa **600 request** trong một phút, vượt quá sẽ bị chặn).
    - `TG_CHAT_ID`: ID nhóm Telegram của bạn.
    - `TG_BOT_TOKEN`: Token của bot Telegram (cần thêm bot vào nhóm Telegram).
    - `IP_WHITE_LIST`: Danh sách các IP được phép (nếu có).

3. **Thiết lập crontab** để chạy script tự động, ví dụ:

    ```sh
    * * * * * for i in {1..6}; do /usr/bin/python3 /path/to/autorl.py & sleep 10; done
    ```

## Demo

### Trên Telegram:

![Demo Telegram](./demo.png)

### Trên Cloudflare:

![Demo Cloudflare](./demo-cf.png)

## Lưu ý

- **IP bị chặn sẽ không được tự động mở chặn**.
- Nếu logrotate không được cấu hình đúng, việc phân tích toàn bộ tệp `access.log` có thể tiêu tốn nhiều tài nguyên hệ thống.
- **AutoRL không lưu trữ mẫu tấn công**, do đó chúng ta không thể biết chính xác cách cuộc tấn công diễn ra.

---

Nếu có bất kỳ vấn đề gì hoặc cần hỗ trợ, hãy để lại bình luận hoặc liên hệ qua Telegram! 🚀