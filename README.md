# Cisco Switch – Основна конфигурация (Cheat Sheet)

## Режими (по промпта се познава къде си)

| Промпт | Режим | Как влизаш |
|---|---|---|
| `Switch>` | User EXEC (само гледане) | по подразбиране |
| `Switch#` | Privileged EXEC (пълен достъп) | `enable` |
| `Switch(config)#` | Global config | `configure terminal` |
| `Switch(config-line)#` | Line config (console/VTY) | `line console 0` или `line vty 0 15` |

Излизане: `exit` (1 ниво назад) · `end` (директно до `#`)

---

## 1. Име на устройството

```
Switch# configure terminal
Switch(config)# hostname Sw-Floor-1
Sw-Floor-1(config)#
```

Само сменя името в промпта. Не влияе на работата.

---

## 2. Паролите – 3 "врати", 3 различни ключалки

Мисли си за switch-а като за сграда с 3 входа. Всеки вход се заключва отделно:

| Врата | За какво е | Команда за парола |
|---|---|---|
| **Console** | Физически кабел директно в switch-а | `line console 0` → `password ...` |
| **Privileged EXEC** | Преминаване от `>` към `#` (админ права) | `enable secret ...` |
| **VTY** | Отдалечен достъп през мрежата (Telnet/SSH) | `line vty 0 15` → `password ...` |

### 2.1 Console – физически достъп

```
Sw-Floor-1# configure terminal
Sw-Floor-1(config)# line console 0
Sw-Floor-1(config-line)# password cisco
Sw-Floor-1(config-line)# login
Sw-Floor-1(config-line)# end
```

- `password cisco` – задава паролата
- `login` – **включва проверката**. Без него паролата се игнорира!

### 2.2 Enable secret – най-важната парола

```
Sw-Floor-1# configure terminal
Sw-Floor-1(config)# enable secret class
Sw-Floor-1(config)# exit
```

- Пази достъпа до `#` режима, откъдето можеш да променяш всичко.
- Използвай **`enable secret`**, не `enable password`. Secret се пази хеширана (силна защита), а `enable password` е в чист текст.
- Тук няма `login` – проверява се автоматично.

### 2.3 VTY – отдалечен достъп

```
Sw-Floor-1# configure terminal
Sw-Floor-1(config)# line vty 0 15
Sw-Floor-1(config-line)# password cisco
Sw-Floor-1(config-line)# login
Sw-Floor-1(config-line)# end
```

- `0 15` = 16 виртуални линии = до 16 едновременни отдалечени сесии.
- Същата логика като при console: `password` + `login`.

### Разликата с една дума

- **Console** – стоиш пред switch-а с кабел.
- **VTY** – свързваш се от разстояние по мрежата.
- **Enable secret** – независимо как си влязъл (console или VTY), това е втората ключалка преди админ правата.

---

## 3. Криптиране на паролите в конфига

Проблем: `password cisco` стои в `show running-config` като чист текст. Всеки, който погледне през рамо или види конфига, я чете.

```
Sw-Floor-1# configure terminal
Sw-Floor-1(config)# service password-encryption
```

Проверка:

```
Sw-Floor-1# show running-config
line con 0
 password 7 094F471A1A0A
 login
line vty 0 4
 password 7 094F471A1A0A
 login
line vty 5 15
 password 7 094F471A1A0A
 login
```

- `7` пред хеша = слабо криптиране (Type 7). Лесно се разчита с онлайн инструменти.
- Целта е само да не се четат паролите на пръв поглед в конфига.
- **Не** защитава паролите по мрежата, само във файла.
- Затова `enable secret` е по-силна: тя използва хеш, който не се връща назад.

### Накратко: 3 вида запис на парола

| Вид | Как изглежда | Сигурност |
|---|---|---|
| Чист текст (`password cisco`) | `password cisco` | Никаква |
| Type 7 (`service password-encryption`) | `password 7 094F...` | Слаба, обратима |
| `enable secret` (хеш) | `secret 5 $1$...` (или по-силен тип) | Силна, необратима |

---

## 4. Banner (предупредително съобщение)

```
Sw-Floor-1# configure terminal
Sw-Floor-1(config)# banner motd #Authorized Access Only#
```

- Показва се на всеки, който опита да се свърже.
- `#` е **разделител** – маркира началото и края на текста. Може да е всеки символ, който не се среща в самия текст.
- Целта е правна: да е ясно, че неоторизиран достъп е забранен. Не пиши "Welcome".

---

## 5. Running-config vs Startup-config

Switch-ът има **два** файла с конфигурация:

| | `running-config` | `startup-config` |
|---|---|---|
| Къде е | RAM | NVRAM |
| Какво е | Текущата конфигурация, която работи **сега** | Запазената конфигурация, която се зарежда при стартиране |
| Промени | Прилагат се **веднага** | Не се променя сама |
| При рестарт / изключване | **Изчезва** (RAM е volatile) | **Остава** |

Всяка команда, която въвеждаш, влиза само в `running-config`. Ако не запазиш, при рестарт всичко се губи.

### Преглед

```
Sw-Floor-1# show running-config
Building configuration...
Current configuration : 1351 bytes
!
! Last configuration change at 00:01:20 UTC Mon Mar 1 1993
!
version 15.0
no service pad
service timestamps debug datetime msec
service timestamps log datetime msec
service password-encryption
!
hostname Sw-Floor-1
!
(output omitted)
```

```
Sw-Floor-1# show startup-config
```

Сравни двете, за да видиш има ли незапазени промени.

### Запазване

```
Sw-Floor-1# copy running-config startup-config
```

(кратко: `wr`)

### Рестарт – reload

```
Sw-Floor-1# reload
System configuration has been modified. Save? [yes/no]: yes
Proceed with reload? [confirm]
```

- Рестартира switch-а. При стартиране зарежда `startup-config` в RAM.
- Ако има незапазени промени, пита дали да ги запише. `yes` = запазва, `no` = губи ги.
- `confirm` се потвърждава с **Enter**.

### Връщане към запазената конфигурация (отмяна на грешки)

Объркал си нещо и **не си запазил**? Рестартирай и откажи запазването:

```
Sw-Floor-1# reload
System configuration has been modified. Save? [yes/no]: no
```

Switch-ът се връща към последния `startup-config`.

### Нулиране до фабрични настройки

```
Sw-Floor-1# erase startup-config
Sw-Floor-1# reload
```

- `erase startup-config` изтрива запазената конфигурация.
- След `reload` switch-ът стартира празен.
- На switch-ове е добре да изтриеш и VLAN базата: `delete vlan.dat` (преди `reload`).

> Запомняне: **running = RAM = живо, но временно**. **startup = NVRAM = запазено**.
> Двете команди `show` се пишат от `#` (privileged EXEC) режим.

---

## 6. Отдалечен достъп: SVI vs VTY

### Какво е всяко

| | SVI (`interface vlan 1`) | VTY (`line vty`) |
|---|---|---|
| Какво е | Виртуален интерфейс с **IP адрес** на switch-а | Виртуални **терминални линии** (сесии) |
| Отговаря на въпроса | **На кой адрес** се свързвам? | **През коя сесия** влизам и как се удостоверявам? |
| Какво настройваш | IP адрес, маска, `no shutdown` | Парола/потребител, `login`, `transport input` |
| Аналогия | Адресът на сградата | Входните врати вътре в сградата |

Layer 2 switch-ът по принцип няма IP адрес. SVI му го дава, за да може да се управлява по мрежата. Без SVI **няма** remote достъп, колкото и да е настроен VTY. Без VTY SVI е просто адрес, на който никой не отваря.

### Flow (как стига връзката до CLI)

```
Твоят компютър (192.168.1.10)
        |
        |  ssh admin@192.168.1.2
        v
SVI - interface vlan 1 (192.168.1.2)     <- връзката стига до switch-а
        |
        v
VTY линии (line vty 0 4)                 <- сесията се приема, проверява се login
        |
        v
CLI на switch-а (SW1>)                   <- въвеждаш команди
```

### Кое кога се ползва

| Ситуация | Какво ползваш |
|---|---|
| Стоиш пред switch-а с кабел | **Console** (не изисква IP, SVI или VTY) |
| Switch-ът е нов/празен, няма IP | **Console** (първоначална настройка) |
| Управляваш от разстояние по мрежата | **SVI** (адрес) + **VTY** (сесия) |
| Нужна е сигурна връзка | VTY със **SSH** (`transport input ssh`) |
| Лаб без значение за сигурност | VTY с Telnet (парола + `login`, виж секция 2.3) |

Console работи винаги. Remote достъпът изисква предварителна настройка през Console.

### Стъпки за SSH достъп

**1. Влез през Console** (кабел, PuTTY или Packet Tracer)

**2. SVI: даваш IP на switch-а**

```
enable
configure terminal
interface vlan 1
 ip address 192.168.1.2 255.255.255.0
 no shutdown
 exit
```

**3. Потребител и SSH ключове**

```
hostname SW1
ip domain-name lab.local
username admin secret MyPass123!
crypto key generate rsa
```

- Питат ли за размер на ключа: `2048`.
- RSA ключ не се генерира без `hostname` (различно от `Switch`) и `ip domain-name`.

**4. VTY: разрешаваш SSH и вход с потребител**

```
ip ssh version 2
line vty 0 4
 login local
 transport input ssh
 exit
```

- `login local` = проверява `username`/`secret`, а не обща парола на линията.
- `transport input ssh` = приема само SSH (Telnet е блокиран).

**5. Default gateway** (само ако компютърът е в друга мрежа)

```
ip default-gateway 192.168.1.1
```

**6. Проверка на switch-а**

```
show ip interface brief
show ip ssh
show running-config
```

- `Vlan1` трябва да е **up/up**.
- Ако е `down/down` или `up/down`: провери дали има активен порт във VLAN 1 (свързан кабел).

**7. Свързване от компютъра** (с IP в същата мрежа, напр. `192.168.1.10/24`)

```
ssh -l admin 192.168.1.2
```

**8. Запази**

```
copy running-config startup-config
```

### Често объркване

- `password` + `login` на VTY (секция 2.3) е най-простият вариант (Telnet), с обща парола.
- `login local` + `username` е по-добрият вариант и е нужен за SSH.
- VTY `0 4` е 5 сесии, `0 15` е 16. И двете са валидни, зависи от модела.
- Работи ли `ping 192.168.1.2`, но SSH не: проблемът е във VTY/SSH настройките. Не работи ли и ping: проблемът е в SVI, кабела или IP адреса на компютъра.

---

## Бърз чеклист

```
configure terminal
hostname Sw-Floor-1
enable secret class
line console 0
 password cisco
 login
 exit
line vty 0 15
 password cisco
 login
 exit
service password-encryption
banner motd #Authorized Access Only#
end
copy running-config startup-config
```

> `cisco` и `class` са примерни пароли за лаб. В реална мрежа ползвай дълги и уникални.
