# MQTT controller for Beok BOT-313 thermostat
### Замена WiFi модуля Broadlink в термостате Beok BOT-313 WiFi на ESP8266

   WiFi термостаты Beok вполне заслуженно ценятся "умнодомостроителями" и давно зарекомендовали себя как 
функциональные, отработанные и надежные устройства. Они имеют неплохой внешний вид и несколько дизайнов на любой вкус, 
удобный интерфейс и гибкую систему работы по расписанию в автоматическом режиме.

   Штатное приложение умеет управлять термостатами в локальной сети (без выхода наружу) либо через пропиретарные облачные скревера.
Но, к сожалению, к самому приложению [Beok Home](https://play.google.com/store/apps/details?id=com.beok.heat) как раз есть претензии.

   Кроме того рано или поздно возникает задача интегрировать термостат в локальную систему управления 
типа MajorDoMo или HomeAssistant. А поскольку WiFi часть этих термостатов построена на базе стандартного модуля Broadlink - 
эта задача решается с помощью установки соответствующей интеграции, например [модуля Broadlink для MajorDoMo](https://connect.smartliving.ru/addons/category1/32.html) + [SBeokThermostat](https://connect.smartliving.ru/addons/category6/245.html)). К сожалению, эти модули периодически опрашивают состояние всех зарегистрированных устройств, что создает дополнительную нагрузку на сервер и приводит к сушественным задержкам при отрабатывании сценариев "по событию". Особенно это заметно если часть термостатов по какой-то причине находится в офлайне.

Хотелось бы упростить интеграцию в локальные системы умного дома, обеспечив им прямой доступ к термостату по протоколу MQTT. Это позволит не только разгрузить сервер умного дома, отказавшись от модуля интеграции, но и реализовать независимые от сервера сценарии управления на уровне протокола MQTT.

Как было отмечено выше, удаленное управление термостаом реализовано на базе пропиретарного WiFi модуля Broadlink BL3335-P (описание можно найти в [Docs](https://github.com/mosave/Beok2MQTT/tree/main/Docs)), который взаимодействует с контроллером термостатоа через последовательный интерфейс (9600/None/1). К сожалению, BL335-P имеет закрытую прошивку и инструменты разработки. Однако анализ данных, передаваемых по uart показал, что для управления MCU используется тот же протокол, что уже реализован в модуле Broadlink для MajorDoMo. Поэтому оказалось достаточно просто заменить модуль BL335 на ESP8266 и превратить термостат в полноценное клиентское MQTT устройство. Еще одним бонусом при этом является и возможность встраивания в термостат дополнительных сенсоров (датчик влажности, освещенности или движения).
 
### Использованное железо

  * Термостат: 
    * [Beok BOT-313 WiFi с любым суффиксом (от $25)](http://www.beok-controls.com/pro_view.asp?id=66) ([проверено](https://aliexpress.ru/item/4000202232813.html))
    * [Beok TGP-51 WiFi с любым суффиксом ($35)](http://www.beok-controls.com/pro_view.asp?id=82) ([проверено](https://aliexpress.ru/item/4000695450163.html))
    * Все термостаты, совместимые с модулем [Broadlink для MajorDoMo в режиме "PHP only"](https://connect.smartliving.ru/addons/category1/32.html).
    * С большой вероятностью любой [WiFi термостат Beok](http://www.beok-controls.com/product.asp). Но здесь, понятно, никаких гарантий дать нельзя.
  * ESP-01S или любой другой ESP8266 модуль (от $10/10шт)
  * Дополнительный датчик влажности [GY-213V-HTU21D](https://www.aliexpress.com/item/32482508209.html) ($11/10шт)

### Прошивка WiFi контроллера

В каталоге [BOT313Firmware](https://github.com/mosave/Beok2MQTT/tree/main/BOT313Firmware) находятся исходники вполне рабочей прошивки, которая тем не менее все еще находится в разработке. В основе прошивки лежит [IoT Framework](https://github.com/mosave/AELib) и поэтому (соответственно) поддерживает все основные команды управления устройством, реализованные в фреймворке.

### Список MQTT топиков и соответствующих им команд для взаимодействия с термостатом

 * **Power**: Состояние термостата. 1 - включен, 0 - выключен
   * **SetPower**: Включить (1) или выключить (0)
 * **RoomTemp**: Текущая температура воздуха (точность 0.5°)
 * **FloorTemp**: Текущая температура внешнего датчика, если поддерживается и установлен
 * **FloorTempMax**: Максимальная температура разаргрева пола (при наличии датчика температуры пола)
 * **TargetTemp**: Текущая целевая температура
   * **SetTargetTemp**: Установка целевой температуры
 * **TargetTempMax**..**TargetTempMin**: Диапазон допустимых значений целевой температуры

 * **AutoMode**: Включен автоматический (1) либо ручной (0) режим управления
   * **SetAutoMode**: Выбор автоматического (1) либо ручного (0) режима управления

 * **Sensor**: Конфигурация датчиков температуры (зависит от модели термостата, см. документацию)
   * **SetSensor**: Выбор конфигурации датчиков температуры

 * **Heating**: Состояние реле управления обогревателем. 1 - обогреватель включен, 0 - выключен
 * **Locked**: Блокировка кнопок: 1 - включена, 0 - выключена
   * **SetLocked**: Включить (1) или выключить (0) блокировку кнопок
 * **TargetSetManually**: В режиме автоматического управления целевая температура была изменена кнопками (1/0)

 * **LoopMode**: Текущий способ выбора первого или второго расписания в автоматическом режиме: 0: 5+2 дня, 1: 6+1 день, 2: 7 дней (см. документацию к термостату)
   * **SetLoopMode**: Установка способа выбора расписания, 0, 1 или 2 

 * **Schedule**, **Schedule2**: Первое и второе расписания в режиме автоматической работы. Указывает 
   с какого времени требуется установить заданную температуру. В первом расписании определяется 5 временных интервалов,
   во втором расписании только 2 (см. документацию к термостату)
   * **SetSchedule**, **SetSchedule2**: Установка расписаний работы. Формат соответствует значению топиков **Schedule** и **Schedule2**

 * **Hysteresis**: Точность, с которой будет выдерживаться заданная температура
 * **AdjTemp**: Поправка к значению датчика температуры RoomTemp/FloorTemp уже учитывают эту поправку.
   * **SetAdjTemp**: Задание поправки значения датчика температуры
 * **AutoAdjMode**: Режим автоматического управления поправкой **AdjTemp**. Доступен только при использовании дополнительного 
   встроенного цифрового датчика температуры и влажности и управляет температурой воздуха (**Sensor**=0). Доступные режимы: 
     * 0: AdjTemp задается только вручную в настройках термостата;
     * 1: AdjTemp подбирается так, чтобы RoomTemp совпадал с температурой цифрового датчика (топик **Sensor/Temperature**)
     * 2: AdjTemp подбирается так, чтобы RoomTemp совпадал с субъективно ощущаемой температурой (топик **Sensor/HeatIndex**)
   * **SetAutoAdjMode**: Установка режима автоматического управления текущего веремени и дня недели. Допустимый формат времени "ЧЧ:ММ" или "ЧЧ:ММ:СС". В случае если в Config.h температуры и влажности.
 * **AntiFroze**: Режим защиты от замерзания включен (1) либо выключен (0)
 * **PowerOnMemory**: Восстанавливать (1) или не восстанавливать (0) состояние термостата после пропадения питания
 * **Time**, **Weekday**: Время (ЧЧ:ММ) и текущий день недели (1..7)
   * **SetTime**, **SetWeekday**: Задание текущего веремени и дня недели. Допустимый формат времени "ЧЧ:ММ" или "ЧЧ:ММ:СС". В случае если в Config.h определена константа 
     TIMEZONE - время будет автоматически синхронизироваться по первому доступному NTP серверу в следующем порядке: "адрес MQTT брокера", "time.google.com" и "time.nist.gov".


### Фотографии

[Несколько фотографий можно найти здесь](https://github.com/mosave/Beok2MQTT/tree/main/Photos)

