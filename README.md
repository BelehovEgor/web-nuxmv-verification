# Веб-сервер 
Веб-сервер обслуживает запросы клиентов. Число потоков сервера зафиксировано (1-3). Когда приходит запрос от клиента - он ставится в очередь. Когда доходит очередь до клиента - он обслуживается от 1 до 3 тактов. Длина очереди зафиксирована (1-5).  Максимальное число клиентов зафиксировано (от 5 до 8).
Если клиент не обслуживается довольно долго, то происходит ошибка времени ожидания, если сервер не может обслужить клиента, то он может сразу отказать в обслуживании. 
Необходимо разработать алгоритм обработки клиентских запросов, при котором у клиентов не случается ошибка времени ожидания, все клиенты справедливо получают обслуживание и т.п. Для сравнения необходимо разработать вторую процедуру, которая не обеспечивает эти свойства. 

# Решение

## Статусная модель клиента
+ **NOT_CREATED** - клиент не создан, стартовое состояние
+ **NEW** - клиент готов создать запрос - в этот момент он может попытаться встать в очередь
+ **PROCESSING** - клиент попал на обработку к определенному потоку
+ **PROCESSED** - запрос клиента был полностью обработан
+ **TIMEOUT** - клиент слишком долго ждал
+ **CANCELLED** - система понимает, что не способна обработать еще один запрос

## Другие важные поля
+ **queue** - очередь в которую записываются клиенты - встают в конец и продвигаются в ячейки обработки (0 или 0-1 в зависимости от числа потоков)
+ **times** - опциональный массив подсчета времени - необходимость заключается в проверке, что нет ситуации, когда состояние клиента не меняется, а время идет
+ **thRemains**,	**thClients** - пара переменных для потоков - номер обрабатываемого клиента и кол-во затраченного времени. ALARM - нарочно не вынесена в другой модуль - иначе потоки могут игнорироваться, работать без нагрузки и тп.

## Ход поиска решения
Выполнялась работа инкрементами, попытка написать сразу все с 0 обернулась провалом. По готовым реализациям можно увидеть ветки для разного числа потоков и длины очереди, также появилась версия со счетчиком времени,
который предоставляет новый путь в TIMEOUT. Любые другие расширения данных решений ведут к очень большому времени ожидания при выполнении запуска программы.

## Результаты или как сломать верификацию

### Правила проверки

Правило присущее только большему числу потоков чем 1 - потоки не обрабатывают одно и тоже
```
CTLSPEC 
	AG
	(
		(thClients[0] != -1 -> thClients[0] != thClients[1]) &
		(thClients[1] != -1 -> thClients[0] != thClients[1])
	);
```
Если клиент был создан - он завершит обработку
```
CTLSPEC 
	AG
	(
		(states[0] = NEW -> EF (states[0] = PROCESSED | states[0] = CANCELLED)) &
		(states[1] = NEW -> EF (states[1] = PROCESSED | states[1] = CANCELLED)) &
		(states[2] = NEW -> EF (states[2] = PROCESSED | states[2] = CANCELLED)) &
		(states[3] = NEW -> EF (states[3] = PROCESSED | states[3] = CANCELLED))
	);
```
Клиент никогда не должен попасть в таймаут
```
CTLSPEC 
	AG
	(
		(states[0] != NOT_CREATED -> (states[0] != TIMEOUT)) & 
		(states[1] != NOT_CREATED -> (states[1] != TIMEOUT)) & 
		(states[2] != NOT_CREATED -> (states[2] != TIMEOUT)) & 
		(states[3] != NOT_CREATED -> (states[3] != TIMEOUT))
	);

```

### Как сломать
Как говорилось выше есть 2 серьезных отличия реализации - с подсчетом времени и без.
В случае подсчета времени у нас есть переходы во все возможные состояния, поэтому чтобы случился таймаут необходимо - отключить возможность отенять запросы и (или) уменьшить максимальное время ожидания.
В случае бесподсчетного решения - переход в состоянии отмененного перевести в состояния таймаута - т.к. очередь уже вся занята время ожидания 100% будет больше требуемого и конечный статус известен.

В качестве примеров выводов программы

### Успешный запуск
```
PS D:\itmo\verify\nuXmv-2.0.0-win64.tar\nuXmv-2.0.0-win64\bin> ./nuXmv.exe '.\async\api_2_thread_async_v2_3q_try_count_time.smv'
*** This is nuXmv 2.0.0 (compiled on Mon Oct 14 18:05:39 2019)
*** Copyright (c) 2014-2019, Fondazione Bruno Kessler
*** For more information on nuXmv see https://nuxmv.fbk.eu
*** or email to <nuxmv@list.fbk.eu>.
*** Please report bugs at https://nuxmv.fbk.eu/bugs
*** (click on "Login Anonymously" to access)
*** Alternatively write to <nuxmv@list.fbk.eu>.

*** This version of nuXmv is linked to NuSMV 2.6.0.
*** For more information on NuSMV see <http://nusmv.fbk.eu>
*** or email to <nusmv-users@list.fbk.eu>.
*** Copyright (C) 2010-2019, Fondazione Bruno Kessler

*** This version of nuXmv is linked to the CUDD library version 2.4.1
*** Copyright (c) 1995-2004, Regents of the University of Colorado

*** This version of nuXmv is linked to the MiniSat SAT solver.
*** See http://minisat.se/MiniSat.html
*** Copyright (c) 2003-2006, Niklas Een, Niklas Sorensson
*** Copyright (c) 2007-2010, Niklas Sorensson

*** This version of nuXmv is linked to MathSAT
*** Copyright (C) 2009-2019 by Fondazione Bruno Kessler
*** Copyright (C) 2009-2019 by University of Trento and others
*** See http://mathsat.fbk.eu

WARNING *** Processes are still supported, but deprecated.      ***
WARNING *** In the future processes may be no longer supported. ***

WARNING *** The model contains PROCESSes or ISAs. ***
WARNING *** The HRC hierarchy will not be usable. ***
-- specification AG ((thClients[0] != -1 -> thClients[0] != thClients[1]) & (thClients[1] != -1 -> thClients[0] != thClients[1]))  is true
-- specification AG ((((states[0] = NEW -> EF (states[0] = PROCESSED | states[0] = CANCELLED)) & (states[1] = NEW -> EF (states[1] = PROCESSED | states[1] = CANCELLED))) & (states[2] = NEW -> EF (states[2] = PROCESSED | states[2] = CANCELLED))) & (states[3] = NEW -> EF (states[3] = PROCESSED | states[3] = CANCELLED)))  is true
-- specification AG ((((states[0] != NOT_CREATED -> states[0] != TIMEOUT) & (states[1] != NOT_CREATED -> states[1] != TIMEOUT)) & (states[2] != NOT_CREATED -> states[2] != TIMEOUT)) & (states[3] != NOT_CREATED -> states[3] != TIMEOUT))  is true
```

### Сломанный запуск
```
PS D:\itmo\verify\nuXmv-2.0.0-win64.tar\nuXmv-2.0.0-win64\bin> ./nuXmv.exe '.\async\api_2_thread_async_v2_3q.smv'
*** This is nuXmv 2.0.0 (compiled on Mon Oct 14 18:05:39 2019)
*** Copyright (c) 2014-2019, Fondazione Bruno Kessler
*** For more information on nuXmv see https://nuxmv.fbk.eu
*** or email to <nuxmv@list.fbk.eu>.
*** Please report bugs at https://nuxmv.fbk.eu/bugs
*** (click on "Login Anonymously" to access)
*** Alternatively write to <nuxmv@list.fbk.eu>.

*** This version of nuXmv is linked to NuSMV 2.6.0.
*** For more information on NuSMV see <http://nusmv.fbk.eu>
*** or email to <nusmv-users@list.fbk.eu>.
*** Copyright (C) 2010-2019, Fondazione Bruno Kessler

*** This version of nuXmv is linked to the CUDD library version 2.4.1
*** Copyright (c) 1995-2004, Regents of the University of Colorado

*** This version of nuXmv is linked to the MiniSat SAT solver.
*** See http://minisat.se/MiniSat.html
*** Copyright (c) 2003-2006, Niklas Een, Niklas Sorensson
*** Copyright (c) 2007-2010, Niklas Sorensson

*** This version of nuXmv is linked to MathSAT
*** Copyright (C) 2009-2019 by Fondazione Bruno Kessler
*** Copyright (C) 2009-2019 by University of Trento and others
*** See http://mathsat.fbk.eu

WARNING *** Processes are still supported, but deprecated.      ***
WARNING *** In the future processes may be no longer supported. ***

WARNING *** The model contains PROCESSes or ISAs. ***
WARNING *** The HRC hierarchy will not be usable. ***
-- specification AG ((thClients[0] != -1 -> thClients[0] != thClients[1]) & (thClients[1] != -1 -> thClients[0] != thClients[1]))  is true
-- specification AG ((((states[0] = NEW -> EF (states[0] = PROCESSED | states[0] = CANCELLED)) & (states[1] = NEW -> EF (states[1] = PROCESSED | states[1] = CANCELLED))) & (states[2] = NEW -> EF (states[2] = PROCESSED | states[2] = CANCELLED))) & (states[3] = NEW -> EF (states[3] = PROCESSED | states[3] = CANCELLED)))  is true
-- specification AG ((((states[0] != NOT_CREATED -> states[0] != TIMEOUT) & (states[1] != NOT_CREATED -> states[1] != TIMEOUT)) & (states[2] != NOT_CREATED -> states[2] != TIMEOUT)) & (states[3] != NOT_CREATED -> states[3] != TIMEOUT))  is false
-- as demonstrated by the following execution sequence
Trace Description: CTL Counterexample
Trace Type: Counterexample
  -> State: 1.1 <-
    states[-1] = NOT_CREATED
    states[0] = NOT_CREATED
    states[1] = NOT_CREATED
    states[2] = NOT_CREATED
    states[3] = NOT_CREATED
    queue[0] = -1
    queue[1] = -1
    queue[2] = -1
    thRemains[0] = 0
    thRemains[1] = 0
    thClients[0] = -1
    thClients[1] = -1
    threadCount = 2
    testClientCount = 4
    queueMaxSize = 3
    c0.isClientInQueue = FALSE
    c0.isQ2Free = TRUE
    c0.isQ1Free = TRUE
    c0.isQ0Free = TRUE
    c0.state = NOT_CREATED
    c1.isClientInQueue = FALSE
    c1.isQ2Free = TRUE
    c1.isQ1Free = TRUE
    c1.isQ0Free = TRUE
    c1.state = NOT_CREATED
    c2.isClientInQueue = FALSE
    c2.isQ2Free = TRUE
    c2.isQ1Free = TRUE
    c2.isQ0Free = TRUE
    c2.state = NOT_CREATED
    c3.isClientInQueue = FALSE
    c3.isQ2Free = TRUE
    c3.isQ1Free = TRUE
    c3.isQ0Free = TRUE
    c3.state = NOT_CREATED
  -> Input: 1.2 <-
    _process_selector_ = c1
    running = FALSE
    c3.running = FALSE
    c2.running = FALSE
    c1.running = TRUE
    c0.running = FALSE
  -> State: 1.2 <-
    states[1] = NEW
    c1.state = NEW
  -> Input: 1.3 <-
  -> State: 1.3 <-
    queue[2] = 1
    c0.isQ2Free = FALSE
    c1.isClientInQueue = TRUE
    c1.isQ2Free = FALSE
    c2.isQ2Free = FALSE
    c3.isQ2Free = FALSE
  -> Input: 1.4 <-
  -> State: 1.4 <-
    queue[0] = 1
    c0.isQ0Free = FALSE
    c1.isQ0Free = FALSE
    c2.isQ0Free = FALSE
    c3.isQ0Free = FALSE
  -> Input: 1.5 <-
    _process_selector_ = c2
    c2.running = TRUE
    c1.running = FALSE
  -> State: 1.5 <-
    states[2] = NEW
    queue[2] = -1
    thClients[0] = 1
    c0.isQ2Free = TRUE
    c1.isQ2Free = TRUE
    c2.isQ2Free = TRUE
    c2.state = NEW
    c3.isQ2Free = TRUE
  -> Input: 1.6 <-
  -> State: 1.6 <-
    states[-1] = NEW
    queue[0] = -1

    queue[2] = 2
    thRemains[0] = 1
    thClients[0] = -1
    c0.isQ2Free = FALSE
    c0.isQ0Free = TRUE
    c1.isClientInQueue = FALSE
    c1.isQ2Free = FALSE
    c1.isQ0Free = TRUE
    c2.isClientInQueue = TRUE
    c2.isQ2Free = FALSE
    c2.isQ0Free = TRUE
    c3.isQ2Free = FALSE
    c3.isQ0Free = TRUE
  -> Input: 1.7 <-
  -> State: 1.7 <-
    queue[0] = 2
    c0.isQ0Free = FALSE
    c1.isQ0Free = FALSE
    c2.isQ0Free = FALSE
    c3.isQ0Free = FALSE
  -> Input: 1.8 <-
    _process_selector_ = c1
    c2.running = FALSE
    c1.running = TRUE
  -> State: 1.8 <-
    states[-1] = NOT_CREATED
    queue[2] = -1
    thClients[0] = 2
    c0.isQ2Free = TRUE
    c1.isQ2Free = TRUE
    c2.isQ2Free = TRUE
    c3.isQ2Free = TRUE
  -> Input: 1.9 <-
  -> State: 1.9 <-
    queue[2] = 1
    c0.isQ2Free = FALSE
    c1.isClientInQueue = TRUE
    c1.isQ2Free = FALSE
    c2.isQ2Free = FALSE
    c3.isQ2Free = FALSE
  -> Input: 1.10 <-
  -> State: 1.10 <-
    queue[1] = 1
    c0.isQ2Free = TRUE
    c0.isQ1Free = FALSE
    c1.isQ2Free = TRUE
    c1.isQ1Free = FALSE
    c2.isQ2Free = TRUE
    c2.isQ1Free = FALSE
    c3.isQ2Free = TRUE
    c3.isQ1Free = FALSE
  -> Input: 1.11 <-
    _process_selector_ = c0
    c1.running = FALSE
    c0.running = TRUE
  -> State: 1.11 <-
    states[0] = NEW
    queue[2] = -1
    thClients[1] = 1
    c0.state = NEW
  -> Input: 1.12 <-
  -> State: 1.12 <-
    states[-1] = NEW
    queue[1] = -1
    queue[2] = 0
    thRemains[1] = 1
    thClients[1] = -1
    c0.isClientInQueue = TRUE
    c0.isQ2Free = FALSE
    c0.isQ1Free = TRUE
    c1.isClientInQueue = FALSE
    c1.isQ2Free = FALSE
    c1.isQ1Free = TRUE
    c2.isQ2Free = FALSE
    c2.isQ1Free = TRUE
    c3.isQ2Free = FALSE
    c3.isQ1Free = TRUE
  -> Input: 1.13 <-
    _process_selector_ = c1
    c1.running = TRUE
    c0.running = FALSE
  -> State: 1.13 <-
    queue[1] = 0
    c0.isQ2Free = TRUE
    c0.isQ1Free = FALSE
    c1.isQ2Free = TRUE
    c1.isQ1Free = FALSE
    c2.isQ2Free = TRUE
    c2.isQ1Free = FALSE
    c3.isQ2Free = TRUE
    c3.isQ1Free = FALSE
  -> Input: 1.14 <-
    _process_selector_ = c3
    c3.running = TRUE
    c1.running = FALSE
  -> State: 1.14 <-
    states[-1] = NOT_CREATED
    states[3] = NEW
    queue[2] = -1
    thClients[1] = 0
    c3.state = NEW
  -> Input: 1.15 <-
    _process_selector_ = c1
    c3.running = FALSE
    c1.running = TRUE
  -> State: 1.15 <-
    queue[2] = 1
    c0.isQ2Free = FALSE
    c1.isClientInQueue = TRUE
    c1.isQ2Free = FALSE
    c2.isQ2Free = FALSE
    c3.isQ2Free = FALSE
  -> Input: 1.16 <-
    _process_selector_ = c3
    c3.running = TRUE
    c1.running = FALSE
  -> State: 1.16 <-
    states[3] = TIMEOUT
    c3.state = TIMEOUT

```
