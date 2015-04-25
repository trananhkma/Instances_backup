# Instances_backup
<i>Giải pháp dự phòng cho các máy ảo trên hệ thống OpenStack</i>
###I. Thông tin LAB
**Ý tưởng:** Kết hợp [**Rsync**](https://github.com/hocchudong/rsync) và [**GlusterFS**](https://github.com/hocchudong/GlusterFS) để thực hiện dự phòng. Sử dụng rsync đẩy thư mục chứa máy ảo trên Compute1 và Compute2 xuống ***replicate volume*** tạo trên GFS1 và GFS2. Thực hiện vào 4h, 12h và 19h hàng ngày.

![Mô hình LAB](https://github.com/trananhkma/image/blob/master/dgfdg.png)

###II. Các bước triển khai
####1. Tạo replicate volume trên GFS
**Cài đặt GlusterFS 3.6 trên cả hai server GFS:**

    apt-add-repository ppa:gluster/glusterfs-3.6
    apt-get update
    apt-get install glusterfs-server -y

Trên cả hai server này đã có phân vùng **vdb1**, được mount vào **/gluster/vdb1/**. <br>
**Cấu hình trên một trong hai GFS server:**
Ở đây tôi chọn GFS2 (10.10.10.196).<br>
Thêm node GFS1 vào trusted pool:

    gluster probe 10.10.10.195

Tạo replicate volume như sau:

    gluster volume create instance-vol rep 2 transport tcp 10.10.10.195:/gluster/vdb1/backup-195 10.10.10.196:/gluster/vdb1/backup-196

Tiếp theo mount volume đã tạo vào một folder để sử dụng, ở đây tôi chọn /root/backup:

    mount -t glusterfs 10.10.10.196:/instance-vol /root/backup

Tạo 2 thư mục chứa lần lượt cho Compute1 và Compute2:

    cd /root/backup
    mkdir compute1 compute2

####2. Cấu hình Rsync và Crontab
Ý tưởng là viết script thực hiện đẩy file bằng rsync, đặt lịch với sự hỗ trợ của crontab.<br>
Trên GFS2, tạo SSH key để thực hiện ssh đến Compute1 và Compute2 mà không cần password:

    ssh-keygen

Nhấn enter 3 lần (Đặt đường dẫn chứa key mặc định và **không** đặt passphrase)<br>
Đẩy key đến Compute1, gán quyền cho key:

    ssh uvdc@10.10.10.101 mkdir -p .ssh
    cat .ssh/id_rsa.pub | ssh uvdc@10.10.10.101 'cat >> .ssh/authorized_keys'
    ssh uvdc@10.10.10.101 "chmod 700 .ssh; chmod 640 .ssh/authorized_keys"

Làm tương tự với Compute2:

    ssh uvdc@10.10.10.102 mkdir -p .ssh
    cat .ssh/id_rsa.pub | ssh uvdc@10.10.10.102 'cat >> .ssh/authorized_keys'
    ssh uvdc@10.10.10.102 "chmod 700 .ssh; chmod 640 .ssh/authorized_keys"

Sau khi hoàn thành bước này có thể ssh đến Compute1 và Compute2 mà không cần nhập password.<br>
Tiếp theo tạo script lấy dữ liệu dùng rsync:

    vim /root/backup.sh

Chèn nội dung sau:

    #!/bin/bash
    
    COM1="10.10.10.101"
    COM2="10.10.10.102"
    SRC="/var/lib/nova/instances/"
    DES1="/root/backup/compute1"
    DES2="/root/backup/compute2"
    
    rsync -avz --exclude=_base/ -e ssh uvdc@$COM1:$SRC $DES1
    rsync -avz --exclude=_base/ -e ssh uvdc@$COM2:$SRC $DES2

Thêm quyền thực thi:

    chmod +x /root/backup.sh

Cấu hình Crontab để chạy script trên vào 4h, 12h, 19h hàng ngày:

    crontab -e

Thêm dòng sau:

    0 4,12,19 * * * /root/backup.sh

Save lại rồi khởi động lại crontab.<br>
<br>
Cấu hình trên cả Compute1 và Compute2:<br>
Gán quyền sở hữu thư mục cho user uvdc:

    sudo chown -R uvdc:uvdc /var/lib/nova/instances/

Hoàn thành bài LAB.
