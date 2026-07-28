# hello-world
第一个新仓库
123456 hello
1.创建 Docker 网络
docker network create sonarnet
2.启动 PostgreSQL 容器
docker run -d --name postgres \
  -e POSTGRES_USER=sonar \
  -e POSTGRES_PASSWORD=sonar \
  -e POSTGRES_DB=sonarqube \
  --network sonarnet \
  postgres:13

3.启动 SonarQube 容器
docker run -d --name sonarqube \
  -restart unless-stopped \
  -p 9000:9000 \
  -e SONAR_JDBC_URL=jdbc:postgresql://postgres:5432/sonarqube \
  -e SONAR_JDBC_USERNAME=sonar \
  -e SONAR_JDBC_PASSWORD=sonar \
  --network sonarnet \
  sonarqube:latest
docker ps

docker logs sonarqube
4.docker stop sonarqube
5.docker start sonarqube
6.docker restart sonarqube
