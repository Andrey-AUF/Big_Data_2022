# Подключение к виртуальной среде и установка
## 1 ШАГ
Нужно воспользоваться файлом docker-compose.yml для загрузки образа. Там прописывается конфигурация установки. 
## 2 ШАГ
С помощью команды docker-compose up --build -d 
установим образ.
## 3 ШАГ
Далее подключимся к серверу: ssh root@localhost -p 2222. 
## 4 ШАГ
Откроем директорию cd /home/mapr/lab_1. 
## 5 ШАГ
После в данной директории выполним команды: 
1) echo 'export PATH=$PATH:/opt/mapr/spark/spark-3.2.0/bin' > /root/.bash_profile
2) source /root/.bash_profile
3) apt-get update && apt-get install -y python3-distutils python3-apt
4) wget https://bootstrap.pypa.io/pip/3.6/get-pip.py
5) python3 get-pip.py
6) pip install jupyter
7) pip install pyspark
8) jupyter-notebook --ip=0.0.0.0 --port=50001 --allow-root --no-browser
Последняя команда нужна для запуска Jupyter Notebook. Лабораторную будем выполнять там. 


Скрин работы контейнера в docker:
![image](https://user-images.githubusercontent.com/70701437/209532995-e3f96179-f2d6-4181-9bd5-0bbc6f2ac9c0.png)

