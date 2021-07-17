# 04 hadoop-HDFS集群搭建-HA模式概念

- 伪分布式：在一个节点启动所有的角色：NN，DN，SNN

- 完全分布式：
  - 基础环境
  - 部署配置
    - 角色在哪里启动
      - NN：core-site.xml: fs.defaultFS hdfs://node01:9000
      - DN: slaves node01
      - SNN: hdfs-site.xml: dfs.namenode.secondary.http.address node01:50090
    - 角色启动时的细节配置
      - dfs.namenode.dir
      - dfs.datanode.data.dir
  - 初始化和启动
    - 格式化
      - FSImage
      - version
    - start-dfs.sh
      - 加载配置文件
      - 通过ssh免密启动相应的角色
- 伪分布式到完全分布式：重新规划角色
- 

