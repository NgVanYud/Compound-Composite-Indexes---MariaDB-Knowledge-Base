[Source](https://mariadb.com/kb/en/library/compound-composite-indexes/ "Permalink to Compound (Composite) Indexes - MariaDB Knowledge Base")

# Compound (Composite) Indexes - Kiến thức căn bản về MariaDB

## Một bài học nhỏ trong "compound indexes" ("chỉ số kết hợp")

This document starts out trivial and perhaps boring, but builds up to more interesting information, perhaps things you did not realize about how MariaDB and MySQL indexing works.
Tài liệu này mở đầu bằng những thứ có vẻ tầm thường và nhàm chán, nhưng để tạo lên những thông tin thú vị, có lẽ bạn đã không nhận ra những điều về cách mà MariaDB và MySQL đánh index làm việc như nào.
Điều này cũng giải thích [EXPLAIN][1] (tới một mức độ nào đó).

(Phần lớn được áp dụng vào những loại csdl không phải của MySql.)

## Bàn luận về truy vấn ( truy vấn để bàn luận)

Câu hỏi là "Khi nào Andrew Johnson làm tổng thống của nước Mỹ?".

Bảng `Presidents` có sẵn như này:
    
    
    +-----+------------+----------------+-----------+
    | seq | last_name  | first_name     | term      |
    +-----+------------+----------------+-----------+
    |   1 | Washington | George         | 1789-1797 |
    |   2 | Adams      | John           | 1797-1801 |
    ...
    |   7 | Jackson    | Andrew         | 1829-1837 |
    ...
    |  17 | Johnson    | Andrew         | 1865-1869 |
    ...
    |  36 | Johnson    | Lyndon B.      | 1963-1969 |
    ...
    

("Andrew Johnson" được chọn cho bài học này vì nó có nhiều bản sao.)

Chỉ mục nào sẽ là tốt nhất cho câu hỏi này? Cụ thể hơn là, những gì sẽ là tốt nhất cho câu truy vấn này:
    
    
        SELECT  term
            FROM  Presidents
            WHERE  last_name = 'Johnson'
              AND  first_name = 'Andrew';
    

Thử một số Index...

* No indexes (*Không index*)
* INDEX(first_name), INDEX(last_name) (two separate indexes) (*Phân thành 2 chỉ mục*)
* "Index Merge Intersect" (*Hợp nhất các chỉ mục*)
* INDEX(last_name, first_name) (a "compound" index) 
* INDEX(last_name, first_name, term) (a "covering" index) 
* Variants (*Các biến thể*)

## Không đánh chỉ mục

Cũng tốt thôi, tôi nói linh tinh 1 chút ở đây. Tôi có một PRIMARY Key ở `seq`, nhưng nó lại không có nhiều lợi ích cho câu truy vấn mà chúng ta đang tìm hiểu ( nghiên cứu).
    
    
    mysql>  SHOW CREATE TABLE Presidents G
    CREATE TABLE `presidents` (
      `seq` tinyint(3) unsigned NOT NULL AUTO_INCREMENT,
      `last_name` varchar(30) NOT NULL,
      `first_name` varchar(30) NOT NULL,
      `term` varchar(9) NOT NULL,
      PRIMARY KEY (`seq`)
    ) ENGINE=InnoDB AUTO_INCREMENT=45 DEFAULT CHARSET=utf8
    
    mysql>  EXPLAIN  SELECT  term
                FROM  Presidents
                WHERE  last_name = 'Johnson'
                  AND  first_name = 'Andrew';
    +----+-------------+------------+------+---------------+------+---------+------+------+-------------+
    | id | select_type | table      | type | possible_keys | key  | key_len | ref  | rows | Extra       |
    +----+-------------+------------+------+---------------+------+---------+------+------+-------------+
    |  1 | SIMPLE      | Presidents | ALL  | NULL          | NULL | NULL    | NULL |   44 | Using where |
    +----+-------------+------------+------+---------------+------+---------+------+------+-------------+
    
    # Or, using the other form of display:  EXPLAIN ... G
               id: 1
      select_type: SIMPLE
            table: Presidents
             type: ALL        <-- Implies table scan
    possible_keys: NULL
              key: NULL       <-- Implies that no index is useful, hence table scan
          key_len: NULL
              ref: NULL
             rows: 44         <-- That's about how many rows in the table, so table scan
            Extra: Using where
    

## Triển khai chi tiết

Đầu tiên, hãy mô tả cách mà InnoDB lưu trữ và sử dụng các chỉ mục.
* Dữ liệu và PRIMARY KEY là các "clustered" cùng nằm trên 1 Btree.
* Truy vấn trên Btree thì khá nhanh và hiệu quả. Đối với bảng có hàng triệu bản ghi được chia thành 3 cấp độ của Btree, và 2 cấp độ cao nhất có thể được lưu trữ trong cache.
* Với mỗi chỉ mục phụ trong 1 BTree khác, với một PRIMARY KEY ở tại nút lá.
* Thu thập các item 'liên tục' (theo các chỉ mục) từ 1 Btree rất là hiệu quả vì chúng được lưu trữ 1 cách liên tục.

* Vì mục đích đơn giản, chúng ta có thể đếm được mỗi truy vấn trên BTree như 1 đơn vị của công việc, và bỏ qua các lần quét cho những mục tiếp theo. Cái này gần xấp xỉ bằng số lần truy cập vào ổ đĩa trên 1 bảng lớn trong một hệ thống bận rộn.

Với MyISAM, một PRIMARY KEY sẽ không lưu trữ dữ liệu, và vì vậy hãy coi nó như là khoá phụ(quá đơn giản).

## INDEX(first_name), INDEX(last_name)

Với người mới làm quen, khi học về cách lập chỉ mục, và quyết định lập chỉ mục cho nhiều cột cùng một lúc. Nhưng...

MySQL lại hiếm khi sử dụng nhiều hơn một chỉ mục cùng một lúc trong câu truy vấn, Vì vậy, nó sẽ tự phân tích các chỉ mục có thể thực hiện.

* first_name -- ở đây có 2 row có thể dùng được (khi có một truy vấn Btree, chúng quét liên tục) 
* last_name -- ở đây cũng có 2 row có thể dùng được, giả sử nó lấy last_name. Ở đây là các bước thực hiện lệnh SELECT: 

1\. Sử dụng INDEX(last_name), tìm 2 chỉ mục có mục với last_name = 'Johnson'. 

2\. Lấy PRIMARY KEY (được thêm ngầm vào mỗi chỉ mục phụ trong InnoDB); lấy giá trị (17, 36). 

3\. Tiếp cận dữ liệu bằng cách sử dụng seq = (17, 36) để lấy những dòng cho Andrew Johnson và Lyndon B. Johnson. 

4\. Sử dụng phần còn lại của mệnh đề WHERE để lọc ra những hàng mong muốn từ tất cả các kết quả. 
3. 
5\. Gửi lại câu trả lời (1865-1869). 
    
    
    mysql>  EXPLAIN  SELECT  term
                FROM  Presidents
                WHERE  last_name = 'Johnson'
                  AND  first_name = 'Andrew'  G
      select_type: SIMPLE
            table: Presidents
             type: ref
    possible_keys: last_name, first_name
              key: last_name
          key_len: 92                 <-- VARCHAR(30) utf8 may need 2+3*30 bytes
              ref: const
             rows: 2                  <-- Two 'Johnson's
            Extra: Using where
    

## "Hợp nhất các chỉ mục"

Tốt thôi, nếu bạn thực sự thông minh và quyết định MySQL có đủ thông minh để sử dụng cùng 1 lúc 2 tên chỉ mục để lấy câu trả lời. Cái này gọi là "giao điểm".
 
 1\. Sử dụng INDEX(last_name), tìm 2 chỉ mục có mục với last_name = 'Johnson'; lấy (7, 17) 
 
 2\. Sử dụng INDEX(first_name), tìm 2 chỉ mục có mục với first_name = 'Andrew'; lấy (17, 36) 
 
 3\. "And" (Hợp nhất) hai danh sách lại với nhau là (7,17) và (17,36) = (17) 
 
 4\.Tiếp cận dữ liệu bằng cách sử dụng seq = (17) để lấy các row cho Andrew Johnson. 
 
 5\. Gửi lại câu trả lời (1865-1869).
    
    
               id: 1
      select_type: SIMPLE
            table: Presidents
             type: index_merge
    possible_keys: first_name,last_name
              key: first_name,last_name
          key_len: 92,92
              ref: NULL
             rows: 1
            Extra: Using intersect(first_name,last_name); Using where
    

The EXPLAIN fails to give the gory details of how many rows collected from each index, etc.

## INDEX(last_name, first_name)

Cái này có thể gọi là chỉ mục "hợp chất" hoặc "hỗn hợp" từ khi nó có nhiều hơn một cột.
 
 1\. Drill down the BTree for the index to get to exactly the index row for Johnson+Andrew; get seq = (17). 
 
 2\. Reach into the data using seq = (17) to get the row for Andrew Johnson. 
 
 3\. Gửi lại các câu trả lời (1865-1869). Thế này tốt hơn. Trong thực tế thì đây có thể là cách "tốt nhất".
    
    
        ALTER TABLE Presidents
            (drop old indexes and...)
            ADD INDEX compound(last_name, first_name);
    
               id: 1
      select_type: SIMPLE
            table: Presidents
             type: ref
    possible_keys: compound
              key: compound
          key_len: 184             <-- The length of both fields
              ref: const,const     <-- The WHERE clause gave constants for both
             rows: 1               <-- Goodie!  It homed in on the one row.
            Extra: Using where
    

## "Bao đóng": INDEX(last_name, first_name, term)

Ngạc nhiên chưa! Chúng ta hoàn toàn có thể làm nó tốt hơn 1 chút. Một chỉ mục "bao đóng" là một trong tất cả các trường của mệnh đề SELECT có thể tìm thấy được. Nó được thêm vào mà không cần tiếp cận các dữ liệu để hoàn thành nhiệm vụ.
 
 1\. Đào sâu xuống dưới của BTree để xác định các dòng chỉ mục 1 cách chính xác cho Johnson+Andrew; lấy seq = (17). 
 
 2\. Gửi lại câu trả lời (1865-1869). Cái "dữ liệu" BTree không được đụng vào; đây là một trong những cải tiến so với "hợp nhất".
    
    
        ... ADD INDEX covering(last_name, first_name, term);
    
               id: 1
      select_type: SIMPLE
            table: Presidents
             type: ref
    possible_keys: covering
              key: covering
          key_len: 184
              ref: const,const
             rows: 1
            Extra: Using where; Using index   <-- Note
    

Mọi thứ đề tương tự như khi sử dụng "hợp nhất", ngoại trừ việc bổ sung thêm "sử dụng các chỉ mục".

## Các biến thể

* Điều gì sẽ xảy ra nếu bạn xáo trộn các trường trong mệnh đề WHERE?Câu trả lời là: Thứ tự của các phần được AND với nhau không quan trọng. 
* Điều gì sẽ xảy ra nếu bạn xáo trộn các trường trong mệnh đề Index? Câu trả lời là: Nó có thể tạo ra những thay đổi đáng kể. Hơn cái một chút nhiều. 
* Điều gì sẽ xảy ra nếu bổ sung thêm các trường ở cuối? Câu trả lời là: Tác hại khá nhỏ; cũng có thể có rất nhiều (ví dụ như, 'covering'). 
* Dư thừa? Với điều này, nếu bạn có cả 2 cái sau: INDEX(a), INDEX(a,b)? Câu trả lời là: Việc dư thừa chi phí trên mệnh đề INSERTs; nó rất hiếm khi hữu ích đối với SELECTs. 
* Tiền tố? Với vấn đề này, INDEX(last_name(5). first_name(5)) Câu trả lời là: Không cần bận tâm; nó hiếm khi có ích, và thường gây hại hơn. (Chi tiết sẽ được đề cập trong 1 chủ đề khác.) 

## Thêm các ví dụ:
    
    
        INDEX(last, first)
        ... WHERE last = '...' -- good (even though `first` is unused)
        ... WHERE first = '...' -- index is useless
    
        INDEX(first, last), INDEX(last, first)
        ... WHERE first = '...' -- 1st index is used
        ... WHERE last = '...' -- 2nd index is used
        ... WHERE first = '...' AND last = '...' -- either could be used equally well
    
        INDEX(last, first)
        Both of these are handled by that one INDEX:
        ... WHERE last = '...'
        ... WHERE last = '...' AND first = '...'
    
        INDEX(last), INDEX(last, first)
        In light of the above example, don't bother including INDEX(last).
    

## Postlog

Refreshed -- Oct, 2012; more links -- Nov 2016