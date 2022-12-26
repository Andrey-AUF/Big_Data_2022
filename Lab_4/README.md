## 1
1) Напишем класс Philosopher, в нем будет содержаться информация о соседях и соседских вилках.
```python
class Philosopher(Process):
	def __init__(self, task_name: str, id: int, fork_path: str, eat_seconds: int = 15, max_id: int = 5):
		super().__init__()
		self.root = task_name
		self.fork = fork_path
		self.id = id
		self.left_fork_id = id
		self.right_fork_id = id + 1 if id + 1 < max_id else 0
		self.eat_seconds = eat_seconds
		self.partner_id_left = id - 1 if id - 1 >= 0 else max_id-1
		self.partner_id_right = id + 1 if id + 1 < max_id else 0
```
2) Стол будет блокироваться (более общий процесс) и будут блокироваться соседние вилки(менее общий процесс), так как только один философ может кушать, остальные будут думать.
```python
table_lock = zk.Lock(f'{self.root}/table', self.id)
left_fork = zk.Lock(f'{self.root}/{self.fork}/{self.left_fork_id}', self.id)
right_fork = zk.Lock(f'{self.root}/{self.fork}/{self.right_fork_id}', self.id)
```
3) У каждого философа будет время, в течении которого он может кушать.
```python
start = time()
while time() - start < self.eat_seconds:
```
4) Блокируем вилки если они свободны, и философ поел не больше, чем его соседи.
```python
with table_lock:
	if len(left_fork.contenders()) == 0 and len(right_fork.contenders()) == 0 \
		and counters[self.partner_id_right] >= counters[self.id] \
		and counters[self.partner_id_left] >= counters[self.id]:
			left_fork.acquire()
			right_fork.acquire()
```
5)Если вилки заблокированы, то философ кушает, потом он освободит вилки и сообщит об этом, счетчик количества еды в философе увеличиваем при этом. Если он не кушает, вероятнее он думает.
```python
if left_fork.is_acquired and right_fork.is_acquired:
	print(f'{self.id} Философ: Возьму-ка немного вот этого...')
	counters[self.id] += 1
	sleep(MEAL_TIME_SEC)
	left_fork.release()
	right_fork.release()
else:
	print(f'{self.id} Философ: Что бы мне покушать...')
	sleep(WAITING_TIME_SEC)
```
6) Задаем константы и создаем процессы с философами.
```python
root = '/task1'
fork_path = 'forks'
seconds_eat = 25
WAIT_EAT_MS = 1.25
WAIT_AFTER_ALL_DONE = 0.4

master_zk = KazooClient()
master_zk.start()

if master_zk.exists(root):
    master_zk.delete(root, recursive=True)

master_zk.create(root)
master_zk.create('/task1/table')
master_zk.create('/task1/forks')
for i in range(0,5):
    master_zk.create(f'/task1/forks/{i}')

counters = Manager().list()
p_list = list()
for i in range(0, 5):
    p = Philosopher(root, i, fork_path, seconds_eat)
    counters.append(0)
    p_list.append(p)
    
for p in p_list: 
    p.start()
```
## 2
1)	Напишем класс в котором будет ID, путь до своего узла и KazooClient.
```python
def __init__(self, root: str, id: int, zk):
	super().__init__()
	self.url = f'{root}/{id}'
	self.root = root
	self.id = id
	self.zk = zk
```
2)	В методе watch_myself происходит обработка действий, которые приходят от Coordinator.
```python
def watch_myself(data, stat):
    if data == ACTION_DISCONNECT:
        print(f'Клиент {self.id} уведомлён о том, что один из участников отключился')
    else:
        if(stat.version == 1):
            sleep(1)
        if stat.version != 0:
            print(f'Клиент {self.id} принял решение от координатора: {data.decode()}')
```
3)	Тут происходит случайный выбор действия и запуск функции watch_myself.
```python
def run(self):      
	self.zk.start()
  value = ACTION_COMMIT if random.random() > 0.5 else ACTION_ROLLBACK
  print(f'Клиент {self.id} запросил {value.decode()}')
  self.zk.create(self.url, value, ephemeral=True)
  datawatcher = DataWatch(self.zk, self.url, watch_myself)

  sleep(WAIT_HARD_WORK_SECONDS)
  self.zk.stop()
  print(f'Клиент {self.id} отключился')
  self.zk.close()
```
4)	Coordinator следит за работой всех потоков и содержит в себе единственное поле timer, который срабатывает по расписанию.
5)	В методе make_decision выбирается действие методом голосования и результат выбора отправляется каждому клиенту.
```python
Coordinator.timer.cancel()
tr_clients = coordinator.get_children('/task_2/transaction')
commit_counter = 0
abort_counter = 0
for client in tr_clients:
    commit_counter += int(coordinator.get(f'/task_2/transaction/{client}')[0] == ACTION_COMMIT)
    abort_counter +=  int(coordinator.get(f'/task_2/transaction/{client}')[0] == ACTION_ROLLBACK)

# Принимает commit только единогласно
final_action = ACTION_COMMIT if commit_counter == number_of_clients else ACTION_ROLLBACK
for client in tr_clients:
    coordinator.set(f'/task_2/transaction/{client}', final_action) 
````
6)	Метод check_clients получает информацию о клиентах и оповещает других, если один из уже подключенных клиентов отсоединился и говорит им завершить свою работу.
```python
def check_clients():
	tr_clients = coordinator.get_children('/task_2/transaction')
  for i in range(len(Coordinator.session_logs)):
      if Coordinator.session_logs[i] is True and str(i) not in tr_clients:
          print("Один участник отключился, всем остальным разослано сообщение с оповещением")
          sleep(0.5)
          Coordinator.timer.cancel()
          for client in tr_clients:
              coordinator.set(f'/task_2/transaction/{client}', ACTION_DISCONNECT)
          sleep(0.5)
          for client in tr_clients:
              zk_list[int(client)].stop()
              zk_list[int(client)].close()
              p_list[int(client)].kill()
          Coordinator.timer.cancel()
          sys.exit()
```
7)	Метод watch_clients устанавливает значения в session_logs при первом подключение клиента, а далее происходит обработка количества клиентов.
```python
@coordinator.ChildrenWatch('/task_2/transaction')
def watch_clients(clients):
    for client in clients:
        Coordinator.session_logs[int(client)] = True

    if len(clients) == 0:
        if Coordinator.timer is not None:
            Coordinator.timer.cancel()
    else:
        if Coordinator.timer is not None:
            Coordinator.timer.cancel()
        Coordinator.timer = threading.Timer(duration, check_clients) 
        Coordinator.timer.daemon = True
        Coordinator.timer.start()

    if len(clients) < number_of_clients:
        print(f'Подключенные клиенты:{clients}')
    elif len(clients) == number_of_clients:
        make_decision()
```
8)	Создаем общий процесс и клиенты.
```python
Coordinator.session_logs = [False] * number_of_clients # Храним, заходил ли уже клиент
coordinator = KazooClient()
coordinator.start()

if coordinator.exists('/task_2'):
    coordinator.delete('/task_2', recursive=True)

coordinator.create('/task_2')
coordinator.create('/task_2/transaction')

Coordinator.timer = None
root = '/task_2/transaction'
       
for i in range(number_of_clients):
    zk_list.append(KazooClient())
    p = Client(root, i, zk_list[-1])
    p_list.append(p)
    p.start()
    sleep(7)
```
