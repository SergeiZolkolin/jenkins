
# Переводим рутину ручного тестирования 1C на рельсы Jenkins-а и ADD

Вы все еще тестируете свои конфигурации 1С вручную? Да вы просто тратите жизнь впустую! Регулярные читатели инфостарта должны уже слышать  ( и не по наслышке знать) про Vanessa Behavior  и его новых отпрысков – ADD и Vannessa Automation.  

Оба  фреймворка – это замечательное воплощение идей удобного тестирования  функциональности на 1С. Мы составляем cценарные тесты(или «фичи»)  на специальном языке gherkin, описывающем поведение пользователя в интерфейсе 1С Предприятия, а затем вручную прогоняем тесты на запускалке – внешней обработке 1С и узнаем, что у нас работает, а что – не очень. Если вы еще не пробовали рай BDD тестирования, то данный туториал будет максимально полезен: мы сразу убъем двух зайцев – на практическом примере узнаем, что это такое и научимся его правильно готовить.

Под правильной готовкой мы будем понимать не запуск тестов вручную (желающим в руки достаточно «плотный» материала про Ванессу), а создание переиспользуемого пайплайна тестирования в Jenkins. Пайплайн, который будет сам автоматически по расписанию запускать тесты. Пайплайн, который не будет ломать вам рабочие базы. Пайплайн, который даст удобный allure отчет. Наконец, пайплайн, который принесет  уверенность в завтрашнем дне! 

Звучит хорошо, не правда ли? Но сбавим градус пафоса, господа…  и перейдем от теории сразу к практике. Все действия будут выполняться под Windows.

Для наших практических экспериментов потребуется следующий софт:
* Jenkins
* One script
* ADD 6.0.0
* Серверная платформа 1С не ниже 8.3.10

На картинках конечный результат будет выглядеть вот так:

```СКРИНШОТЫ```

## 1. Установка GIT

GIT - наверное уже известная всем система контроля верси кода, которая все больше вхожит в жизнь 1С-ков. Она нам потребуется для того, чтобы  дженкинс могу работать со скриптами нашего пайплайна, которые расположены в экспериментальном репозитории https://github.com/ripreal/erp_features.git (данные репозиторий подойдет, чтобы выполнить туториал, но для разворачивания на продакшене рекомендуется завести свой)
1. Скачиваем последний дистрибутив GIT for Windows и устанавливаем

## 2. Установка и настройка Jenkins-а.

Jenkins – бесплатная среда для автоматического запуска всех скриптов нашего пайплайна по расписанию.  Установка  и первичная настройка дженкинса не принесет никаких проблем.
1. Скачиваем дистрибутив JRE 1.8 и устанавливаем
2. Скачиваем последний дистрибутив Jenkins (на момент статьи это 2.141) и устанавливаем. Все настройки оставляем по-умолчанию.
3. Меняем стандартную кодировку дженкинса на UTF-8. Это важный этап, чтобы в веб-интерфейсе дженкинса все русские символы отображались корректно. Для этого добавляем параметр -Dfile.encoding=UTF8 в тег <arguments> в файле Jenkins.xml, расположенном в корневом каталоге установки дженкинса. Итоговая строка должна выглядеть примерно так:

```<arguments>-Xrs -Xmx256m -Dhudson.lifecycle=hudson.lifecycle.WindowsServiceLifecycle -Dfile.encoding=UTF8 -jar "%BASE%\jenkins.war" --httpPort=8991 --webroot="%BASE%\war"</arguments>```

4. Перезапускаем службу Jenkins в диспетчере задач Windows и проверяем, что все прошло хорошо, открывыв веб-интерфейс дженкинса по адресу http://localhost:8991.

![Начальный экран](media/Jenkins_main.png)

## 3. Настройка slave ноды дженкинса

После установки дженкинса мы получаем одну master ноду, запущенную как системный процесс. Но в приличном jenkins обществе принято использовать master ноду только в качестве менеджера slave агентов, а не ка запускалку пайплайнов. Поэтому чтобы не создавать на нее лишнюю нагрузку и не вешать ее в случае какого-либо неоптимального кода, заводятся slave ноды. 

Для нас необходимость второй ноды обсусловлена еще и сугубо утилитарными нуждами - для правильной работы нашего пайплайна требуется нода, запущенная не как служба, а как консольное приложение. Дело в том, что ADD запускается в среде 1С со всем полагающимся графическим интерфейсом. Если мы будем запускать ADD под master нодой, то мы просто не увидим процесс выполнения ADD тестов на нашей машине (хотя это не помешает успешному выполнению тестов) Для простоты развернем slave на той же машине, на которой запущен и master. Итак наши шаги будут следующими:

1. Разрешаем запуск слейв агентов в качестве коснольных прилоежний. Для этого в веб-интерфейсе дженкинса переходим в меню Manage Jenkins => Configure Global Security => Agents и в поле TCP port for JNLP agents меняем переключатель на Fixed и указываем порт, например 10001.
2. Добавляем ноду. Для этого переходим в меню Manage Jenkins => Manage Nodes и в левой командной панели нажимаем New Node, вводим имя, активируем переключатель Permanent Agent и жмем ок.
3. Вводим параметр ноды:
* Name - имя хоста (компьютера)
* Remote root directory - произвольный путь к каталогу, в котором дженкинс будет выполнять пайплайны, например D:\jenkins
* Labels - произвольное имя, по которому будем ссылаться на ноду в пайплайне. Рекомендую ставить такое же, как имя ноды.
* Launch method - выбираем Launch agent via Java Web Start
4. Жмем save
![Конфиг ноды](media/node_settings.png)

Теперь нужно поднять ноду:

* В главном меню дженкинса в левой части должна появится иконка нашей новой ноды.

![Нода в оффлайне](media/node_icon.png)

* Кликаем на нее и смотрим на последнюю строчку. Это и есть командная строка запуска slave ноды. 

![Строка запуска ноды](media/cmd_nodeline.png)

* Копируем командную строку и записываем ее в bat-ик,  заодно скачиваем agent.jar по гиперссылке. Все это ложим в каталог, который мы выделили ранее для slave дженкинса и запускаем bat-ник. Если все сделано правильно, то через пару секунд запустится консольная слейв нода.
![Нода запущена](media/slave_start.png)


## 3. Настройка shared-libraries

Эта удобная функция позволяет писать и складывать скрипты в отдельные библиотеки для их переиспользования в дальнейшем. Мы будем использовать эти библиотеки постоянно.
1. В веб-интерфейсе дженкинса переходим в меню Manage Jenkins => Global Tool Configuration => Global Pipeline Libraries
2.Нажимем Add и заполняем поля:
* Name: shared-libraries
* Default version: master
* Retrieval method: modern SCM
* Source Code Management: Git (не GitHub)
* Poject Repository: https://github.com/ripreal/erp_features.git

## 4. Настройка Allure

Аллюр позволит генерировать красивые отчеты прямо в дженкинсе по результатам  тестирования в ADD.
1. Устаналиваем плагин allure. В веб-интерфейсе дженкинса переходим в меню Manage Jenkins => Manage plugins => Available, ищем в списке Allure и устанавливаем его
2. Устанавливаем сам дистрибутив allure. В веб-интерфейсе переходим Manage Jenkins => Global Tool Configuration => Allure Commandline installations => Add Allure Commandline. Заполняем появившиеся поля следующим образом
* Name: allure
* Label: allure
* Download URL for binary archive: https://dl.bintray.com/qameta/maven/io/qameta/allure/allure-commandline/2.11.0/allure-commandline-2.11.0.zip
* Subdirectory of extracted archive: allure

## 5. Настройка окружения ADD 

Переходим к установке непосредственно самих утилит, нужных для работы вспомогательных административных скриптов и самого инструмента ADD.

1. Скачиваем последний дистрибутив OneScript и устанавливаем.
2. Устанавливаем библиотеку vannessa-runner для OneScript. Для этого выполняем команду:

```opm install vanessa-runner```

3. Таким же образом устанавливаем библиотеку add

4. Также у вас уже должны быть установлены sclcmd (поставляется вместе с MS SQL Server) и  powershell (с включенной политикой беспрепятственного запуска скриптов) 
 
 ## 6. Создание и настройка пайплайна в Jenkins

 Теперь самое интересное - создать и настроить пайплайн, который и будет запускать по расписанию весь процесс сборки, начиная с копирования эталонных баз и заканчивая их тестированием и очисткой.

 1. В Веб-интерфейсе дженкинса переходим в меню New Item, заполняем проозвольное имя в поле Enter an item name (я выбрал erp_features), выбираем тип скрипта - pipeline и нажимаем ОК
 2. В новом окне открывается конфиг пайплайна. Переходим к группе pipeline, в котором мы введем настройки подключения к репозиторию, в котором хранятся все исходники. Заполняем поля следующим образом:
 * Definition - pipeline script from SCM
 * SCM - git
 * Repository URL - https://github.com/ripreal/erp_features.git
3. Все остальные параметры оставляем по-умолчанию и нажимаем Save

## 7. Первый запуск пайплайна

На самом деле мы неполностью настроили конфиг пайплайна, т.к. часть параметров конфига  хранятся в Jenkinsfile - скрипте в удаленном репозитории, который мы указали в п.6. Поэтому чтобы донастроить пайплайн необходимо выполнить пробный запуск. Для этого:
1. В Веб-интерфейсе дженкинса на главной странице переходим в наш пайплайн erp_features
2. В открывшемся окне нажимаем Build в левой командной панели и наблюдаем процесс сборки. Сборка должна будет упасть - это нормально. Зато теперь у нас вместо кнопки Buid появится другая кнопка Build with parameters - это как раз то, что нам нужно.

![Собрать с параметрами](media/Build_params.png)

## 8. Второй запуск пайплайна

Теперь самое время перейти в реальному запуску пайплайна. Для этого в меню с пайплайном erp_features нажимаем кнопку Build with parameters и видим список параметров, которые нам нужно заполнить. Значения вех введенных параметров будут сохранены для следующих запусков пайплайна:

* server1c - 'Имя сервера 1с, по умолчанию localhost
* server1cPort - Порт рабочего сервера 1с. По умолчанию 1540. Не путать с портом агента кластера (1541)
* serverSql - Имя сервера MS SQL. По умолчанию localhost
* admin1cUser - Имя администратора базы тестирования 1с. Должен быть одинаковым для всех баз
* admin1cPwd - Пароль администратора базы тестирования 1C. Должен быть одинаковым для всех баз
* sqlUser - Имя администратора сервера MS SQL. Если пустой, то используется доменная  авторизация
* sqlPwd - Пароль администратора MS SQL.  Если пустой, то используется доменная  авторизация
* templatebases - Список баз для тестирования через запятую. Например work_erp,work_upp
* storages1cPath - Необязательный. Пути к хранилищам 1С для обновления копий баз тестирования через запятую. Число хранилищ (если указаны), должно соответствовать числу баз тестирования. Например D:/temp/storage1c/erp,D:/temp/storage1c/upp
* storageUser - Необязательный. Администратор хранилищ  1C. Должен быть одинаковым для всех хранилищ
* storagePwd - Необязательный. Пароль администратора хранилищ 1c

