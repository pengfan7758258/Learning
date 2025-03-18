# apoc插件地址
[地址](https://github.com/neo4j-contrib/neo4j-apoc-procedures/releases)


# docker运行插件方式
1. 下载和docker pull neo4j:tag版本对应的apoc插件
2. 运行命令：
```bash
docker run --name my_neo4j -d \
  -p 7474:7474 -p 7687:7687 \
  -v /home/pengfan/neo4j/data:/data \
  -v /home/pengfan/neo4j/plugins:/plugins \
  -e NEO4J_dbms_security_procedures_unrestricted=apoc.* \
  -e NEO4J_dbms_security_procedures_allowlist=apoc.* \
  neo4j:5.26.0
```
3. 插件放在第2点`运行命令`的-v plugins挂载的本地路径下
4. 注意：jar插件不需要解压
5. 测试命令：
	1. 查看拆件版本：`RETURN apoc.version() AS version;` 
	2. apoc的帮助文档：`CALL apoc.help('apoc');`