#Tùy chỉnh trong file policy.json

##I. Mục tiêu thực hiện
 - Tạo ra được 1 user "test" với role "test".
 - Cấp quyền cho role "test" trong từng project cụ thể
 - Quyền được cấp trong từng project là do mình tùy chỉnh

##II. Tìm hiểu về file policy.json
###1. Định nghĩa
  - Mỗi 1 dịch vụ như Identity, Compute, Networking hay Stored đều có 1 chính sách kiểm soát truy cập dựa trên vai trò (role base access control). Với mỗi người dùng có thể truy cập đến từng project với những thao tác cụ thể như nào sẽ được định nghĩa trong file policy.json

  - Đường dẫn của file policy.json thường sẽ là:
    /etc/"project"/policy.json

  - Với project là tên của từng project cụ thể ta muốn tùy chính file policy.json

###2. Cấu trúc file:
   File policy.json là file text dưới định dạng json.
    Mỗi 1 chính sách sẽ được định nghĩa trên 1 dòng theo định dạng: 

     	"<alias>"  : "<define alias>"

       		"<target>" : "<rule>"

 -   Alias: đại diện cho 1 tập role

 -  Target: Hay mục tiêu của chính sách còn có thể coi như 1 hành động ("action") đại diện cho lời gọi API như: tạo máy ảo, tạo router....

 -  Rule: Quy định, chỉ ra quyền role nào được phép gán

 -  Ví dụ: ta có role "test" thuộc user "test". Ta định nghĩa xem role đó trong file policy.json có 2 cách như sau
 
Cách 1:

        "test1" : "role: test"
        "identity:create_user" : "rule: test1"


 Cách 2:

       "identity:create_user" : "role: test"    	


 Ta có thể định nghĩa 1 hành động này thông qua alias hoặc định nghĩa trực tiếp. Việc định nghĩa thông qua alias sẽ giúp ta định nghĩa ra 1 rule bao gồm nh role trong đó

 Với những chính sách đc định nghĩa như sau:
            
            "<target>" : ""

Nghĩa là tất cả các user đều được thực thi target này


##III. Bài toán cụ thể:
 - Ta sẽ sử dụng bản OPENSTACK KILO vs Keystone v3 để thử chỉnh sửa với các file policy.json tương ứng
 - Ngoài user admin đã có sẵn, ta sẽ tạo 1 user test với những quyền trong các project cụ thể như sau:



| |Keystone | Glance| Nova | Neutron|
|--------------|-------|------|-------| -----|
| Được phép | Xem list user, Tạo user | Xem image, up image | Xem máy ảo | Tạo router |
| Không được phép | Xóa user, xóa role | Xóa image| Tạo máy ảo | Xóa router, update thông tin|




###Bước 1: Tạo tenant "test"
 	 
 	 keystone tenant-create --name admin --description "Admin Tenant"

###Bước 2: Thực hiện tạo user "test" với câu lệnh: 

	keystone user-create --name admin --pass <pass user> --email <địa chỉ mail>

###Bước 3: Tạo role "test":
	 
    keystone role-create --name test

###Bước 4: add user "test" thực hiện role "test" trong tennat "test"
	 
    keystone user-role-add --user test --tenant test --role test

###Bước 5: Thực hiện chỉnh sửa với keystone:
 	 	
 	 Thực hiện vào file theo đường dẫn /etc/keystone/policy.json

  Tạo 1 alias trên cùng 

     "test": "role:test",

   Sửa dòng phân quyền cho action là tạo user, list user, list tennet với các dòng tương ứng như sau:


    "identity:get_user": "rule:admin_required or rule:test",
    "identity:list_users": "rule:admin_required or rule:test",
    "identity:creat_users": "rule:admin_required or rule:test",

   Với keystone thì file policy.json mặc định tất cả các hành động đều có được cấp với quền admin, tương ứng là alias : admin_required



###Bước 6: Thực hiện chỉnh sửa với glance
   Vào file theo đường dẫn tương ứng:  

    /etc/glance/policy.json

   Với glance ta thực hiện 1 cách cấp quền trực tiếp ko thông qua alias như ở trên


   Add image và list image:

    "add_image": "role:admin or role:test",
    "get_image": "role:admin or role:test",
    "get_images": "role:admin or role:test",

   Với targer delete image ta chỉ để mỗi role là admin
   
    "delete_image": "role: admin",


###Bước 7: Thực hiện chỉnh sửa với nova
Vào file theo đường dẫn: 

    /etc/nova/policy.json

Tạo alias

    "test": "role:test",

   Mặc định với nova mọi user đều có quyền tạo máy ảo, tuy nhiên ta sẽ tùy chỉnh chỉ để user có quyền admin mới có thể tạo ra.

    "compute: create" : "role: admin_or_owner"



###Bước 8: Thực hiện chỉnh sửa với neutron
 Thực hiện vào file theo đường dẫn 

    /etc/neutron/policy.json

 Tạo alias

    "test": "role:test",

Tạo router

    "get_router:ha": "rule: admin_only or rule: test" 
    "create_router": "rule: admin_only or rule: test",
    "create_router:external_gateway_info:enable_snat": "rule: admin_only or rule: test",
    "create_router:ha": "rule: admin_only or rule: test",
    "get_router": "rule: admin_only or rule: test",
    "get_router:distributed": "rule: admin_only or rule: test",

  Với các targer liên quan đến xóa router để rule tương ứng là admin_only



##IV Test kết quả lab
Có 2 cách test:

##C1: cta có thể sử dụng câu lệnh
- Tạo file script biến môi trường:

 
vi test.sh


    export OS_PROJECT_DOMAIN_ID=default
    export OS_USER_DOMAIN_ID=default
    export OS_PROJECT_NAME=test
    export OS_TENANT_NAME=test
    export OS_USERNAME=test
    export OS_PASSWORD=canvietanh
    export OS_AUTH_URL=http://10.10.10.140:35357/v3
    export OS_IDENTITY_API_VERSION=3

Sau đó sử dụng câu lênh:

    source test.sh

Sau đó test các dịch vụ với các câu lện tương ứng
 
Ví dụ như với keystone:
Được phép list user

<img src="http://i.imgur.com/KnGfIwA.png">

List endpoint sẽ lỗi:

<img src="http://i.imgur.com/uuLgHPc.png">

##C2: Có thể đăng nhập vào dashboard của horizon sau đó dùng giao diện:

Xóa image sẽ bị lỗi:

<img src="http://i.imgur.com/Lwu1n0b.png">

 
