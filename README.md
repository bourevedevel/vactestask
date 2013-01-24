vactestask
==========

Тестовое задание

1. Начальные данные.
1.1. Инструменты и окружение.

	- Язык С++ или/и С.
	- Компилятор gcc или VS20xx (хотя не критично если другой).
	- Любой фреймворк, но предпочтительно Qt.
	- Операционная система семейства Windows NT или GNU/Linux.

1.2. Спектрометрическое устройство.
	Устройство измеряет интенсивности некоторой аналитической величины, распределённой по каналам спектрометра. Спектр – последовательность значений интенсивности (в общем случае вещественные числа) для каждого канала. Число каналов для каждого устройства может варьироваться.
	Сигналы управления:
		- Напряжение работы детектора/сенсора измеряемой величины.
		- Коэффициента усиления сигнала.
		- Запустить/остановить накопление спектра.
		- Очистить накопленные интенсивности (обнулить).
	Данные:
		- Количество спектрометрических каналов (от 1 до 255)
		- Чтение интенсивностей по каждому каналу (значение интенсивности лежит в пределах 0-255)
		- Состояние: измерение/простой и т.д.
	Возможные интерфейсы взаимодействия с устройством: RS-232, RS-485, Ethernet (TCP/IP stack, UDP). 
1.2.1. Протокол.
	Протокол взаимодействия с устройством одинаков для всех интерфейсов.
		Передача данных - пакетная. Порядок следования байт для данных - little endian.
		
		Формат пакета: 
			<Marker><Code><Length><Data><CheckCode>
		где:
			<Marker> = 0xFF
				маркер начала пакета (1 байт)
			<Code> = см. далее
				код операции (1 байт)
			<Length>
				длина поля <Data> в байтах (1 байт).
			<Data> 
				данные длины <Length> (если <Length> = 0, то поле отсутствует).
			<CheckCode>
				наивный проверочный код (1 байт) - сумма по модулю 256 всех байт пакета, включая байт маркера, но кроме самого проверочного байта.
			
		Клиент (например некоторое ПО), использующий устройство, производит запросы, формируя пакеты согласно протоколу, и обрабатывает ответные пакеты.
		В случае если пакет был принят успешно (обрабатываются пакеты все успешно) будет выслан ответ, поле <Code> которого будет равно коду запроса с выставленным старшим битом в 1 = (<Code> | 0x80), т.е. это ACK. В противном случае ответа послано не будет.
		В случае команд 0x10, 0x11, 0x12 - в ответном пакете будут содержаться данные.
		Коды команд <Code>:
			0x00 - Очистить накопленный спектр
				<Length> = 0x00
				<Data> = отсутствует
				Query  [->] <Marker> 0x00 0x00 <CheckCode>
				Answer [<-] <Marker> 0x80 0x00 <CheckCode>
			0x01 - Установить режим измерения
				<Length> = 0x01
				<Data> = 
					0x00 - Остановить накопление спектра
					0x01 - Запустить накопление спектра
				Query  [->] <Marker> 0x01 0x01 <Data> <CheckCode>
				Answer [<-] <Marker> 0x81 0x00 <CheckCode>
			0x02 - Установить коэффициент усиления
				<Length> = 0x02
				<Data> = 2 байта, значение коэффициента усиления в диапазоне 0-65535 (беззнаковое целое 2-х байтное).
				Query  [->] <Marker> 0x02 0x02 <Data> <CheckCode>
				Answer [<-] <Marker> 0x82 0x00 <CheckCode>
			0x03 - Установить напряжение детектора
				<Length> = 0x02
				<Data> = 2 байта, значение напряжение детектора. 
					Число в диапазоне 0-65535 (беззнаковое целое 2-х байтное), соответствующее диапазону вольтажа напряжения детектора 500-1500 В.
				Query  [->] <Marker> 0x03 0x02 <Data> <CheckCode>
				Answer [<-] <Marker> 0x83 0x00 <CheckCode>
			0x10 - Запросить кол-во каналов спектрометра.
				Query  [->] <Marker> 0x10 0x00 <CheckCode>
				Answer [<-] <Marker> 0x90 0x01 <Data> <CheckCode>
				<Data> = 1-255
			0x11 - Запросить статусное слово.
				Query  [->] <Marker> 0x11 0x00 <CheckCode>
				Answer [<-] <Marker> 0x91 0x01 <Data> <CheckCode>
				<Data> = байт статусного слова
					биты:
						0   - 1 если идёт накопление, 0 - иначе
						1-7 - зарезервированы
			0x12 - Запросить спектр.
				Query  [->] <Marker> 0x12 0x00 <CheckCode>
				Answer [<-] <Marker> 0x92 <Length> <Data> <CheckCode>
				<Length> = N
					Количество байт данных, равно количеству каналов спектрометра.
				<Data> = <Channel_1_Int> <Channel_2_Int> ... <Channel_N_Int>
					последовательность байт, каждый из которых содержит значение интенсивности в канале (номер канала есть номер байта в последовательности)

2. Задача.
	Разработайте GUI-приложение со следующим функционалом:
	Приложение должно осуществлять управление и получение данных со спектрометрического устройства по ЛЮБОМУ интерфейсу (на выбор или по всем) согласно приведённому протоколу (1.2.1).
			Управление включает в себя:
				- запуск/останов набора спектра;
				- очистка спектра;
				- задание вольтажа и усиления;
				- задание времени экспозиции набора спектра, по истечении которого набор спектра останавливается.
	Приложение должно реализовывать графическое отображение получаемых спектров во время набора с обновлением экрана не реже 1 раза в 5 секунд, а так же сохранять полученные спектры после остановки набора в файл в произвольном формате.
	
3. Задание рекомендуется выполнить следующим образом:
	- имея account на github сделать fork проекта vactestask;
	- выполнить задание в fork-нутом репозитории;
	- сделать pull-request;
	- прислать на указанную в п.4 почту ссылку на pull-request и Ф.И.О. автора для сопоставления.

	Возможны и другие удобные на ваш взгляд варианты с отсылкой нам на почту.

4. Почта:
	bourevestnik.devel.team@gmail.com