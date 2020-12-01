# Web01
## Source
https://drive.google.com/file/d/1tDI-W9BOB_K5ssLrbaJDlgqyX6l1Jzth/view
## Mô tả sơ lược
Trang web cho phép các bạn tạo nick và đăng nhập. Dựa trên sở thích của bạn trang web sẽ suggest để các bạn có thể match với những người khác. Rất thích hợp với các bạn FA  
![FA](/img/alone.jpg)
## Analysis
Source được public và gồm các file `dbconnect.php`, `index.php`, `login.php`, `logout.php`, `match.php`, `profile.php` và `register.php`  
Các file `dbconnect.php`, `index.php` và `logout.php` khá đơn giản và không có gì đặc biệt. Ta đi sâu vào các file còn lại
### File register.php
Có 2 đoạn code php chính là đoạn kiểm tra một user có tồn tại hay không
![reg_1](/img/reg1.jpg)
và đoạn đăng ký một user mới
![reg_2](/img/reg2.jpg)
Ta thấy đoạn kiếm tra một user có tồn tại hay không bị lỗi SQL injection khi truyền thẳng `$_GET["user"]` vào câu query. Tuy nhiên ta bị giới hạn 25 kí tự ở đây, đồng thời server cũng không trả về kết quả của câu `SELECT` mà chỉ trả về user có exists hay không.  
Ở đoạn đăng ký user mới, mặc dù đã dùng hàm [mysqli_real_escape_string](https://www.php.net/manual/en/mysqli.real-escape-string.php) nhưng nếu để ý ta thấy 2 biến `$yob` và `$id_card` khi insert không nằm trong 2 dấu `'`, và khi đó ta vẫn có thể SQL injection vào 2 biến này. Tuy nhiên đời không như là mơ, 2 biến này bị giới hạn độ dài là 4 ký tự :sob:
### File login.php
Đoạn code check login khá đơn giản
![login](/img/login.jpg)
Từ đây ta có thể biết được cột chứa password tên là `fa_dating_password`.
### File profile.php
Trang này sẽ hiện thị thông tin cá nhân lên, bao gồm `name`, `yob` và `interest` và `id_card`. Đồng thời cho phép cập nhật description - sẽ được hiển thị ở trang `match.php`.
### File match.php
Trang sẽ hiển thị ngẫu nhiên một người có chung sở thích với mình. Ngoài ra nếu là `admin` thì sẽ được quyền lưu lại tên và tuổi của một user vào một file bất kỳ.
![match](/img/match.jpg)
## Exploit
Ta thấy nếu ta đăng nhập được vào nick `admin` thì có thể tạo một file có tên bất kỳ với dữ liệu là `name` của một user → tạo một nick có `name` là 
``` PHP
<?php system($_GET["cmd"]);?>
```
và lưu dưới tên `shell.php` là ta sẽ có một web shell trên server.  
Vậy đầu tiên ta cần đăng nhập vào nick `admin` trước.  
Team mình tiếp cận lỗi SQLi ở phần kiểm tra user tồn tại. Với giới hạn 25 ký tự và tên cột là `fa_dating_password` (18 ký tự) nên team mình đã không dùng các cách SQLi thông thường như blind, error-based, ..., thay vào đó tìm cách extract password mà không dùng đến tên cột. Và cách mà team mình sử dụng là kết hợp [local variable của mysql](https://dev.mysql.com/doc/refman/8.0/en/declare-local-variable.html) và lỗi SQLi ở phần đăng ký user mới.  
Ý tưởng là biến câu query check exists thành `SELECT * FROM fa_dating_users WHERE fa_dating_username = 'admin' INTO @var1,@var2,@var3,@var4,@var5,@var6,@var7`, và sau đó biến câu query tạo mới user thành `INSERT INTO fa_dating_users VALUES ('username','password','pin',yob,'interest',@var2,'name')`. Khi đó cột `fa_dating_id_card` sẽ lưu giá trị `@var2` tương ứng với password của `admin`, và ta có thể đọc được ở trang `profile.php`.  
Payload ban đầu của team mình `$_GET["user"]="admin'into@,@x,@,@,@,@,@#"` (vừa đủ 25 ký tự :joy:), `$_POST["id_card"]="@x__"` (với `_` là khoảng cách)    
Tuy nhiên đời ko như là mơ :sob:, team mình không biết cấu trúc database như nào, nhưng sau khi tạo nick xong thì ở `profile.php` không hiện ra gì hết. Mình đoán do cột `fa_dating_id_card` đã được setup sao cho không insert quá 4 ký tự được. Và đây là lúc biến `$yob` có tác dụng. Team mình sẽ dùng biến `$yob` để đẩy `@x` sang cột `fa_dating_name`.  
Và payload cuối cùng:  
`$_GET["user"]="admin'into@,@x,@,@,@,@,@#"`  
`$_POST["yob"]="0, 0"`  
`$_POST["id_card"]="@x)#"`  
Mọi thứ đều vừa khít :heart_eyes:   
Lúc này thì đăng nhập vào và có được md5 của password, tìm được password của `admin` là `handsomeboy`.  
Việc có shell khá đơn giản như mình đã trình bày ở trên, RCE và có flag thôi.
## Tản mạn
Ban đầu nhóm mình tốn khá nhiều thời gian cho việc bypass cái check length `<= 25` do không nhìn thấy chỗ SQLi ở phần tạo user. Gần 4 tiếng đầu diễn ra trong tâm trạng căng thẳng khi mà bên pwn và crypto đều liên tục có điểm về, trong khi web thì dậm chân tại chỗ :cold_sweat:.  
Sau đó thì ngồi đọc kĩ code lại thì tìm ra bug ở `$yob` và `$id_card` thì tâm trạng mới đỡ hơn, và lại hoảng khi payload ban đầu không hoạt động  như mong muốn, may mà lúc đó vẫn giữ được bình tĩnh để suy nghĩ ra việc cột `fa_dating_id_card` bị giới hạn độ dài.  
 Đỉnh điểm là khi team mình đã có shell nhưng không có flag do mình quá ngu linux :sob:. Tốn gần 20p để tìm cách đặt reverse shell và nhận ra máy không mở port ra ngoài, sau đó lúc team `Nupakachi` submit được flag thì con web shell của mình cũng bị mất luôn, lúc đó mới nhận ra máy chủ reset khi có đội submit flag. May mắn là `Nupakachi` không kịp patch và mình đã up shell lên lại. Cũng nói thêm là lúc mình đang tìm cách đặt reverse shell thì có thấy shell của một đội khác nên đã nói [@Lâm](https://github.com/lamnhh) viết sẵn cái patch, mình thì chưa kịp remove cái shell đó thì đã bị sút =)), nhưng cũng nhờ thế mà lúc chiếm lại được Web01 thì đã có sẵn patch trong người rồi. Mặc dù patch vẫn thọt do password của `admin` vẫn là `handsomeboy` nên có `mysqli_real_escape_string` biến `$_GET["user"]` thì vẫn bị up shell, lần patch thứ 2 thì team mình dùng thêm `htmlspecialchars` để escape cái `name`, từ đó không up web shell được nữa, lúc này thì mới an toàn.  
 ## Kết
 Cảm ơn BTC đã tạo một sân chơi bổ ích cho các bạn sinh viên, cảm ơn thầy Duy và và khoa Công nghệ thông tin đã tạo điều kiện cho bọn em được thi SVATTT 2020, cảm ơn các anh chị trong CyberJutsu đã giúp bọn em trong việc luyện tập các challenge