一. 触发器 调用搜索微服务 rest api 同步数据
问题:
	1. gsql需要添加http请求的扩展功能给触发器函数调用
	2. 如果API调用失败如何处理？
	
二. 触发器的实现逻辑
在增删改之后触发
? 那些字段要触发，行级、列级、表级
实现点:
	1.触发器如何获取增删改的数据？
	2.将数据如何发生到到搜索引擎
	

psql添加http扩展
1. 复制http.os
2. 安装sudo apt-get install libcurl4-openssl-dev 开发包




1、创建表

CREATE TABLE userinfo (
  id smallserial PRIMARY KEY,
  name VARCHAR(10),
  city VARCHAR(10)
);

2、触发器函数

CREATE FUNCTION sync_search() RETURNS trigger AS $sync_search$
	DECLARE
	url varchar := 'http://localhost:8809/sync';
	BEGIN
		-- 根据增删改调用post，put，delete方法
        -- 
        -- 使用特殊变量 TG_OP 来得到操作。

        IF (TG_OP = 'DELETE') THEN
            SELECT http_delete(url ||'?id=' || OLD.id);
            RETURN OLD;
        ELSIF (TG_OP = 'UPDATE') THEN
            SELECT http_put(url, json_each_text(row_to_json(NEW)), 'application/json');
            RETURN NEW;
        ELSIF (TG_OP = 'INSERT') THEN
            SELECT http_post(url, json_each_text(row_to_json(NEW)), 'application/json');
            RETURN NEW;
		ELSIF (TG_OP = '') THEN
			SELECT http_delete(url);
            RETURN OLD;
        END IF;
        RETURN NULL; -- 因为这是一个 AFTER 触发器，结果被忽略
	END;
$sync_search$ LANGUAGE plpgsql;

3、创建触发器

CREATE TRIGGER snyc_to_serrch AFTER INSERT OR UPDATE OR DELETE ON userinfo
    FOR EACH ROW EXECUTE PROCEDURE sync_search();
	
注:TRUNCATE 不支持 FOR EACH ROW
CREATE TRIGGER snyc_truncate_to_serrch AFTER TRUNCATE ON userifo
    FOR STATEMENT EXECUTE PROCEDURE sync_search();

4、删除触发器
DROP TRIGGER snyc_to_serrch ON userinfo;
DROP TRIGGER snyc_truncate_to_serrch ON userinfo;

4、修改函数
ALTER FUNCTION sync_search() RETURNS trigger AS $sync_search$
	DECLARE
	url varchar := 'http://localhost:8809/sync';
	BEGIN
		-- 根据增删改调用post，put，delete方法
        -- 
        -- 使用特殊变量 TG_OP 来得到操作。

        IF (TG_OP = 'DELETE') THEN
            SELECT http_delete(url ||'?id=' || OLD.id);
            RETURN OLD;
        ELSIF (TG_OP = 'UPDATE') THEN
            SELECT http_put(url, json_each_text(row_to_json(NEW)), 'application/json');
            RETURN NEW;
        ELSIF (TG_OP = 'INSERT') THEN
            SELECT http_post(url, json_each_text(row_to_json(NEW)), 'application/json');
            RETURN NEW;
		ELSIF (TG_OP = '') THEN
			SELECT http_delete(url);
            RETURN OLD;
        END IF;
        RETURN NULL; -- 因为这是一个 AFTER 触发器，结果被忽略
	END;
$sync_search$ LANGUAGE plpgsql;
