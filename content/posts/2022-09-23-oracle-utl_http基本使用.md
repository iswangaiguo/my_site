---
title: Oracle UTL_HTTP基本使用
date: 2022-09-23T00:36:49+00:00
categories:
  - 编程
---

最近有个需求，需要对已有的业务进行功能扩展，无法进行源码级别的改动，但是需要检测数据库的新增数据变化，并发送短信。

有几个方案都可以做到，比如轮询数据库，建立临时表，与原始表相对比并打上已发送标记等，或者使用触发器将新增的数据直接加入临时表，每次轮询无发送标记的数据，在发送完成后打上标记。

但是这几种方式都不是特别好，耦合比较严重，增加数据库压力，并且无法做到实时性，不过好在Oracle11g提供了发送HTTP请求的包，这时候问题便简单了，只需要创建**insert触发器**，然后将加入的数据取需要的字段发送HTTP请求，由自己的API程序调用短信接口直接发送即可，可不可以通过Oracle网络请求包直接调用短信接口呢，想来也是可以的，但是需要对PL/SQL编程相对熟悉，对于普通程序员只需要还是包一层比较方便，并且可以进行更细粒度的控制。

## 一、创建ACL并分配权限

Oracle使用UTL_HTTP首先需要添加ACL网络访问控制文件

```sql
begin
  dbms_network_acl_admin.create_acl (       -- 创建访问控制文件（ACL）
    acl         => '/sys/acls/http_acl.xml',          -- 文件名称（可以任意命名，与下一致即可）
    description => 'Send HTTP request',           -- 描述
    principal   => 'NEW_USER',                   -- 授权或者取消授权数据库账号，大小写敏感
    is_grant    => TRUE,                    -- 授权还是取消授权
    privilege   => 'connect',               -- 授权或者取消授权的权限列表
    start_date  => null,                    -- 起始日期
    end_date    => null                     -- 结束日期
  );

  dbms_network_acl_admin.add_privilege (    -- 添加访问权限列表项
    acl        => '/sys/acls/http_acl.xml',           -- 刚才创建的acl名称
    principal  => 'NEW_USER',                    -- 授权或取消授权用户
    is_grant   => TRUE,                     -- 与上同
    privilege  => 'resolve',                -- 权限列表
    start_date => null,
    end_date   => null
  );

  dbms_network_acl_admin.assign_acl (       -- 该段命令意思是允许访问acl名为/sys/acls/sendtodata.xml下授权的用户，使用oracle网络访问包，所允许访问的目的主机，及其端口范围。
    acl        => '/sys/acls/http_acl.xml',
    host       => '*',-- ,              -- ip地址或者域名
                                            -- 且建议使用ip地址或者使用域名，若用localhost，当oracle不是安装在本机上的情况下，会出现问题
    lower_port => 9000,                     -- 允许访问的起始端口号 0/null
    upper_port => 9000                   -- 允许访问的截止端口号 9000/null
  );
  commit;
end;
```

查询已创建的ACL

```sql
SELECT any_path FROM resource_view WHERE any_path like '/sys/acls/%.xml';

/sys/acls/ANONYMOUS/ANONYMOUS877f43e8bece40eaa688e529f5d1a227_acl.xml
/sys/acls/OLAP_XS_ADMIN/OLAP_XS_ADMIN184b49c1c2145fb8ba93fc42b32160_acl.xml
/sys/acls/OLAP_XS_ADMIN/OLAP_XS_ADMIN20525cf164a34bcb1e27445be2012ba_acl.xml
/sys/acls/OLAP_XS_ADMIN/OLAP_XS_ADMIN6d72bed5eb4e83a850647e74f4f161_acl.xml
/sys/acls/OLAP_XS_ADMIN/OLAP_XS_ADMIN8ea661a24abb4053916d858e83e9034_acl.xml
/sys/acls/OLAP_XS_ADMIN/OLAP_XS_ADMINabdfbafe6c41e89bcc4a459455866_acl.xml
/sys/acls/all_all_acl.xml
/sys/acls/all_owner_acl.xml
/sys/acls/bootstrap_acl.xml
/sys/acls/http_acl.xml                  --刚创建的ACL密码
/sys/acls/ro_all_acl.xml
/sys/acls/ro_anonymous_acl.xml
```

如果需要修改ACL，可以直接删除重新创建，删除SQL如下

```sql
begin
  dbms_network_acl_admin.drop_acl(
    '/sys/acls/http_acl.xml'
  );
commit;
end;
```

## 二、创建触发器

测试表结构如下:

```sql
create table TEACHER
(
    ID   NUMBER        not null,
    NAME VARCHAR2(200) not null
)
```

创建存储过程主要包括一个请求的**URL**和两个参数**ID**和**NAME**:

```sql
create or replace procedure post_msg(v_url in varchar2, id in varchar2, name in varchar2) is
    req   UTL_HTTP.REQ;
    resp  UTL_HTTP.RESP;
    data varchar2(4000);
    value VARCHAR2(4000); --过小会报PL/SQL: numeric or value error
begin
    data := 'id=' || id || '&name=' || name;
    req := UTL_HTTP.BEGIN_REQUEST(url => v_url, method => 'POST');
    UTL_HTTP.SET_BODY_CHARSET('UTF-8');
    UTL_HTTP.SET_HEADER(req, 'Content-Type', 'application/x-www-form-urlencoded');
    UTL_HTTP.SET_HEADER(req, 'Content-Length', lengthb(data));
    UTL_HTTP.WRITE_TEXT(req, data);
    resp := UTL_HTTP.GET_RESPONSE(req);
    LOOP
        UTL_HTTP.READ_LINE(resp, value, TRUE);
        DBMS_OUTPUT.PUT_LINE(value);
    END LOOP;
    UTL_HTTP.END_REQUEST(req);
    UTL_HTTP.END_RESPONSE(resp);
EXCEPTION
    WHEN UTL_HTTP.END_OF_BODY THEN
        UTL_HTTP.END_RESPONSE(resp);
    WHEN others then
        DBMS_OUTPUT.PUT_LINE(sqlerrm);
END;
```

这里UTL_HTTP有几个**注意事项**

READ_LINE会抛出以下**END\_OF\_BODY**异常，[官方文档][1]解释如下

```text
This procedure reads the HTTP response body in text form until the end of line is reached and returns the output in the caller-supplied buffer. The end of line is as defined in the function read_line of UTL_TCP. The end_of_body exception will be raised if the end of the HTTP response body is reached. Text data is automatically converted from the response body character set to the database character set.
```

Oracle默认一个Session只能发起5个请求，这里需要在**UTL\_HTTP.END\_OF_BODY**里结束请求，否则超出5个请求将会抛出**TO MANY REQUESTS**异常。

扩展业务功能的时候肯定不希望影响已有的功能，所以需要在with others里面**处理掉所有其他异常**，不让其抛出，否则**数据回滚**影响正常业务。

## 三、测试

oracle中插入一条数据 **insert into TEACHER values (215, '111');**

![](https://objects.wangaiguo.com/wp-content/uploads/2022/09/image.png)

RESTFUL API日志：

![](https://objects.wangaiguo.com/wp-content/uploads/2022/09/image-2.png)

[1]: https://docs.oracle.com/database/121/ARPLS/u_http.htm#ARPLS71070
