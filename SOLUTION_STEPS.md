# Как я решал задание по майнеру для Synth Subnet

## Что нужно было сделать

Задача - запустить майнер в 50 подсети Bittensor, который генерирует прогнозы цен для криптовалют. Майнер должен создавать 100 симуляций ценовых путей на 24 часа вперед с интервалом 5 минут для BTC, ETH, XAU и SOL.

## Что я изучил сначала

### Bittensor базис
Быстро разобрался с основными понятиями:
- Subnet - это специализированная подсеть (50 подсеть для Synth)
- Miner - создает контент (в нашем случае прогнозы цен)
- Validator - оценивает качество работы майнеров
- UID - номер в подсети 
- Hotkey/Coldkey - ключи для кошелька
- TAO - токен сети

### Как работает Synth Subnet
Основное:
- Валидаторы каждый час отправляют запросы майнерам
- Майнер должен вернуть 100 симуляций цен на 24 часа (каждые 5 минут = 289 точек)
- Качество оценивается по CRPS метрике
- Если CRPS > 0 по всем активам - задание выполнено

## Установка окружения

### Создание форка и подключение к серверу

Сначала создал форк оригинального репозитория на GitHub:
1. Зашел на https://github.com/mode-network/synth-subnet
2. Нажал Fork -> Create fork  
3. Репозиторий скопировался в мой профиль: https://github.com/aleksei-pr/synth-subnet

Подключился к RunPod серверу по SSH:

```bash
# Получил данные подключения в RunPod панели:

ssh root@157.157.221.29 -p 34135 -i ~/.ssh/id_ed25519
```

Клонировал свой форк репозитория:

```bash
# Перешел в рабочую директорию
cd /workspace

# Клонировал форк (не оригинал!)
git clone https://github.com/aleksei-pr/synth-subnet.git
cd synth-subnet

# Проверил, что клонировался мой форк
git remote -v
# origin  https://github.com/aleksei-pr/synth-subnet.git (fetch)
# origin  https://github.com/aleksei-pr/synth-subnet.git (push)
```

### Настройка Python окружения

```bash
# Создал виртуальное окружение
python3 -m venv bt_venv
source bt_venv/bin/activate

# Установил зависимости
pip install --upgrade pip
pip install -r requirements.txt
```

Архитектура - основные файлы:
- `neurons/miner.py` - главный файл майнера
- `synth/miner/simulations.py` - логика генерации симуляций
- `synth/miner/price_simulation.py` - модели ценовых путей

Формат ответа - список из 100 путей, каждый путь содержит 289 точек с временем и ценой.

## Тестирование базовой модели

Проверил, что базовый майнер работает:

```bash
PYTHONPATH=. python synth/miner/run.py
```

Получил `CORRECT` - значит все установлено правильно.

**Важно**: Нужен `PYTHONPATH=.` иначе Python не найдет модуль `synth`.

Посмотрел на базовую модель в `price_simulation.py` - это геометрическое броуновское движение.

```python
def simulate_single_price_path(current_price, time_increment, time_length, sigma):
    # Просто случайное блуждание с нормальным распределением
    dt = time_increment / 3600  
    num_steps = int(time_length / time_increment)
    
    std_dev = sigma * np.sqrt(dt)
    price_change_pcts = np.random.normal(0, std_dev, size=num_steps)
    
    cumulative_returns = np.cumprod(1 + price_change_pcts)
    cumulative_returns = np.insert(cumulative_returns, 0, 1.0)
    
    price_path = current_price * cumulative_returns
    return price_path
```

## Улучшение модели

1. Авторегрессию - влияние прошлых цен
2. Изменяющуюся волатильность (GARCH-style)
3. t-распределение вместо нормального (жирные хвосты)
4. Асимметрию - падения увеличивают волатильность
5. Разные параметры для каждого актива

### Изменения в коде

Сначала добавил импорт `scipy` в `price_simulation.py`:

```python
from scipy import stats  # Для t-распределения
```

Потом написал улучшенную функцию `simulate_single_price_path_advanced()` со всеми этими фичами. Основная идея:

- Авторегрессия: текущий return зависит от предыдущего
- Волатильность меняется во времени (GARCH-style) 
- Использую t-распределение для жирных хвостов
- Падения цены увеличивают волатильность больше чем рост
- У каждого актива свои параметры (BTC более волатильный чем XAU)

Потом обновил основную функцию `simulate_crypto_price_paths` чтобы использовать новую модель и принимать параметр `asset`.

И в `simulations.py` добавил одну строку чтобы передавать имя актива:

```python
asset=asset,  # передаем название актива в модель
```

Протестировал - получил `CORRECT`.

### Тест для всех активов

Создал дополнительный тест `test_all_assets.py` чтобы проверить работу со всеми активами:

```bash
PYTHONPATH=. python test_all_assets.py
```

Все 4 актива показали `CORRECT` - модель работает правильно.

## Настройка майнера

Теперь нужно было запустить майнер в сети. Сначала установил Node.js и PM2 для управления процессами:

```bash
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs  
sudo npm install pm2 -g
```

Потом создал Bittensor кошелек:

```bash
btcli wallet create --wallet.name miner --wallet.hotkey default
```

Сохранил мнемонические фразы и адреса (это тестовая сеть, поэтому свечу):
- UID в тестовой сети: 124
- Coldkey: 5HeDL1k59j4wXkRkcH5h2b6JDXmgbBhAvhUNXZ3veXiE3tZd  
- Hotkey: 5F4xuLMQ9azo1B72h6BAqWiYmxrq1v7JjqEZUFu8iyk7W3hX

Зарегистрировался в тестовой сети (netuid 247):

```bash
btcli subnet register --wallet.name miner --wallet.hotkey default --network test --netuid 247
```

В репозитории взял конфигурацию `miner.local.config.js`:

```javascript
module.exports = {
  apps: [{
    name: 'miner',
    interpreter: 'python3',
    script: './neurons/miner.py',
    args: '--netuid 247 --logging.debug --subtensor.network test --wallet.name miner --wallet.hotkey default --axon.port 8091',
    env: { PYTHONPATH: '.' }
  }]
};
```

Она уже была настроена правильно для тестовой сети (netuid 247). Запустил майнер:

```bash
source bt_venv/bin/activate
pm2 start miner.local.config.js
pm2 logs miner
```

## Проверка работы

Майнер запустился успешно, получил UID 124. Проверил статус:

```bash
pm2 list  # показывает status: online
```

Проверил валидацию через API:

```bash
curl "https://api.synthdata.co/validation/miner?uid=124"
```

Показывает `{"validated":false,...}` - майнер не получает запросы от валидаторов.

### Добавление стейка

Чтобы сделать майнер привлекательнее для валидаторов, добавил стейк:

```bash
# Перевел TAO с Alice на майнер  
btcli wallet transfer --wallet.name alice --dest 5HeDL1k59j4wXkRkcH5h2b6JDXmgbBhAvhUNXZ3veXiE3tZd --amount 0.01 --network test

# Добавил стейк на майнер
btcli stake add --wallet.name miner --wallet.hotkey default --amount 0.005 --netuid 247 --network test
```

**Результат:** Стейк `5.51 ኡ` добавлен, но валидаторы в тестовой сети все еще неактивны.

## Финальная проверка

Через 25-27 часов нужно проверить CRPS scores для всех активов:

```bash
curl "https://synth.mode.network/validation/scores/latest?asset=BTC&miner_uid=124"
curl "https://synth.mode.network/validation/scores/latest?asset=ETH&miner_uid=124"  
curl "https://synth.mode.network/validation/scores/latest?asset=XAU&miner_uid=124"
curl "https://synth.mode.network/validation/scores/latest?asset=SOL&miner_uid=124"
```

Если CRPS > 0 для всех активов - задание выполнено.

## Что получилось

Майнер работает в тестовой сети с UID 124, использует улучшенную модель временных рядов вместо дефолтного.

### Основные улучшения модели:
1. **Авторегрессия** - влияние прошлых доходностей
2. **Изменяющаяся волатильность** - GARCH-style модель  
3. **t-распределение** - жирные хвосты вместо нормального распределения
4. **Асимметрия** - падения увеличивают волатильность больше
5. **Актив-специфичные параметры** - разные настройки для BTC/ETH/XAU/SOL

### Что нужно для завершения:
- Дождаться CRPS > 0 для всех активов через 25-27 часов


## Обратная связь
какие были сложности при реализации?
- `ModuleNotFoundError` - решается через `PYTHONPATH=.`
- Отсутствие текстовых редакторов на RunPod - использовал `cat` и `echo`
- Получение тестовых TAO - использовал кошелек Alice
- Низкая активность валидаторов в тестовой сети - даже с добавленным стейком майнер не получает запросы
