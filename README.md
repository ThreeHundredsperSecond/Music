# YandexMusic

## Постановка задачи, ТЗ

Обнаружение треков каверов - важная продуктовая задача, которая может значительно улучшить качество рекомендаций музыкального сервиса и повысить счастье наших пользователей. Если мы умеем с высокой точностью классифицировать каверы и связывать их между собой, то можно предложить пользователю новые возможности для управления потоком треков. Например:  
1.	по желанию пользователя можем полностью исключить каверы из рекомендаций;  
2.	показать все каверы на любимый трек пользователя;  
3.	контролировать долю каверов в ленте пользователя.  
В этом хакатоне вам предлагается разработать решение, которое:  
1.	может классифицировать треки по признаку кавер-некавер;  
2.	связывать (группировать) каверы и исходный трек;  
3.	находит исходный трек в цепочке каверов.  

## Доступные данные:  

### Целевые данные:  
•	track_id - уникальный идентификатор трека;  
•	track_remake_type - метка, присвоенная редакторами. Может принимать значения ORIGINAL и COVER;  
•	original_track_id - уникальный идентификатор исходного трека.  

### Метаинформация:  
•	track_id - уникальный идентификатор трека;  
•	dttm - первая дата появления информации о треке;  
•	title - название трека;  
•	language - язык исполнения;  
•	isrc - международный уникальный идентификатор трека;  
•	genres - жанры;  
•	duration - длительность трека  

### Текст песен  
•	track_id - уникальный идентификатор трека;  
•	lyricId - уникальный идентификатор текста;  
•	text - текст трека.  

## Описание решения

Предполагается, что решение будет представлять двустадийную модель и осуществлено следующим алгоритмом:  
1.	В базу всех треков делается запрос "Найти похожее по тексту" к конкретному треку  
2.	С помощью кластеризации выбираются самые похожие  
3.	Накладывается необходимый фильтр на отобранные треки  

Пункт 3 может являться достаточно расплывчатым и может включать в себя много дополнительных параметров, необходимых заказчику. Например:  
3.1) Отбор песен по жанрам  
3.2) Вывод только оригиналов  
3.3) Вывод только каверов, по степени схожести к запрашиваемому треку или к оригиналу  
3.4) Вывод самых близких песен по тексту, в том числе и не каверы  

![alt text](https://github.com/ThreeHundredsperSecond/Music/blob/main/image/%D0%A0%D0%B8%D1%81%201.png)

Работа online с большим количеством данных накладывает на алгоритм серьезные рамки времени работы и поиска. Для пользователя всё должно быть практически мгновенно. Поэтому необходимо подобрать подход с самым хорошим соотношением точности ко времени выполнения.  

## Exploratory data analysis (EDA)

По результатам анализа, было выявлено следующее:  
### Файл **covers.json**:  
1) соотношение треков оригинал/кавер - 6%/94%  

![alt text](https://github.com/ThreeHundredsperSecond/Music/blob/main/image/%D0%A0%D0%B8%D1%81%202.png)  

2) Имеется большое количество пропусков в колонке **original_track_id** (66776 из 71597). Это сильно ограничивает количество данных и усложняет работу с моделями.

### Файл **lyrics.json**:  
1) Во всех столбцах есть повторы, к одним и тем же трекам могут быть приписаны разные тексты. Это может быть связано с различными источниками этих текстов, исполненную несколько раз (например, два альбома или концертная запись) или же просто дубли загрузки  
2) Если посмотреть на длину треков, то видно, что есть сильно короткие и невероятно длинные треки. Это может быть связано как со спецификой треков, так и с возможными ошибками в данных. Медианная длина текста находится в 1165 символах  


![alt text](https://github.com/ThreeHundredsperSecond/Music/blob/main/image/%D0%A0%D0%B8%D1%81%203.png) 

### Файл **meta.json**:  
1) Пропуски присутствуют в языках и **isrc**   
2) Топ 10 жанров :'FOLK', 'LATINFOLK', 'POP', 'ALLROCK', 'ROCK', 'ALTERNATIVE','ELECTRONICS', 'SOUNDTRACK', 'RAP', 'DANCE'  

![alt text](https://github.com/raphael12/YandexMusic/blob/main/images/%D0%A0%D0%B8%D1%81%204.png)  

3) Наиболее популярные языки: Английский, Испанский, Русский  

![alt text](https://github.com/raphael12/YandexMusic/blob/main/images/%D0%A0%D0%B8%D1%81%205.png)  

4) Самое раннее упоминание из столбца ddtm - 08.2009, последнее упоминание - 2023.10, наибольшее упоминание треков приходится на октябрь-ноябрь
5) Большая часть данных, судя по всему, не имеет каверов, треки с первым и единственным упоминанием доминируют в данных, если не брать во внимание случаи одинаковых названий
6) Для столбца **isrc** извлекли признаки, проверили на корректность, существуют треки для которых один **isrc** для разных **track_id**, возможно из-за того что один трек входит в разные альбомы
7) Дата из **isrc** не всегда совпадает с датой в **ddtm**, есть аномалии с данными больше чем 23 год и их нельзя найти в базе

![alt text](https://github.com/raphael12/YandexMusic/blob/main/images/%D0%A0%D0%B8%D1%81%206.png)  


## Подготовка признаков

Из имеющихся данных были сгенерированы дополнительные признаки:  
### **isrc**   
1) *isrc_country_code* - код страны    
2) *isrc_issuer* - код регистранта (эмитента)
3) *isrc_issuer_full* - код страны + код регистранта  
4) *isrc_year* - год присваивания  

### **genre**   
1) был закодирован методом One-Hot Encoder  
2) *count_genres* - количество жанров, подходящих для трека  

### **text**   
1) *count_rows_text* - количество строк  
2) *len_text* - длина текста  

### **title**  
1) *title_year* - год (в некоторых есть такая информация)  
2) *title_cover* - признак, который будет фиксировать наличие слова cover в названии  


## Классификация
Для музыкальных треков из имеющихся данных были созданы 137 новых признаков (129 признаков-жанров) из предыдущего пункта.  
  
По интересующим нас признакам рассчитали матрицу корреляции:

![alt text](https://github.com/raphael12/YandexMusic/blob/main/images/%D0%A0%D0%B8%D1%81%207.png)  

Видно, что есть сильная связь между целевой переменной и признаками: ид. эмитента (сокращенной и полной), код страны, язык, год выпуска).
Присутствует мультиколлинеарность между признаками, сгенерированными из isrc, языком и годом из dttm.
Более корректно эмитента указывать с кодом страны, поэтому признак isrc_issuer будет отброшен.

Были определены наиболее важные признаки для модели с использованием атрибута feature_importances_ и библиотеки shap:

![alt text](https://github.com/raphael12/YandexMusic/blob/main/images/%D0%A0%D0%B8%D1%81%209.png)  
![alt text](https://github.com/raphael12/YandexMusic/blob/main/images/%D0%A0%D0%B8%D1%81%2010.png)  

По полученным результатам была обучена финальная модель:

![alt text](https://github.com/raphael12/YandexMusic/blob/main/images/%D0%A0%D0%B8%D1%81%208.png)  

### Вывод  
Для музыкальных треков из имеющихся данных были созданы 137 новых признаков (129 признаков-жанров), обучена модель на 85% данных с известными значениями столбца 'track_remake_type' и протестирована на оставшихся. Для данной модели не использовалось содержание заголовков и текста песен, а лишь их размер, а также извлекалась возможная полезная информация из заголовков (год и упоминание "cover").
При подборе гиперпараметров модели была поставлена задача добиться максимально качественного поиска оригинала (минимизация ошибки 1-го рода). Также был проведен отбор признаков используя feature_importance и обучена новая модель. Несмотря на то, что метрика улучшилась, до отбора признаков модель совершала меньше ошибок. Метрика качества выбрана AUC-ROC = 0.992 на тестовой выборке. Проведена оценка важности признаков с учётом "попадания в класс". Самыми важными оказались эмитент и длительность трека.
Данная модель корректно работает при условии, что сушествует лишь 2 класса (оригинал и кавер).

## Кластеризация  

В данном разделе будут исследованы подходы для векторизации, кластеризации и поиска ближайших соседей. Будут исследован следующий стек:   
    1)CountVectorizer  
    2)TfidfVectorizer  
    3)PCA  
    4)NearestNeighbors  
    5)Faiss  
    6)SBERT

**Формирование тестовой выборки**: Сюда вошли данные, которые имеют пометку COVER и точно имеют достоверный ответ в остальном датасете. При этом предполагается поиск именно самых похожих вариантов, те модель будет одинакого искать как каверы для оригинала, так и оригинал для кавера.  
В тестовую выборку вошли:  
COVER       220 штук  
ORIGINAL     86 штук  
Из них оригинальных всего 229 шутк  

**Метрика**: accuracy@n. Проверяем есть ли верный ответ в n кандидатах  

Было исследовано 3 способа векторизации текстов: CountVectorizer, TfidfVectorizer и SBERT. С точки зрения метрики и времени обучения/расчёта хорошие результаты показал SBERT c последующим применением инвертированного мультииндекса Faiss. Примерно c такой же точностью отрабатывает связка TfidfVectorizer + PCA(2048) + Faiss(IndexIMIFlat). В целом SBERT получает более информативную матрицу. 

Результаты работы моделей и время поиска представлены в таблице:  

<table>
    <tr>
        <td>Модель кластеризации</td>
        <td>accuracy@5</td>
        <td>accuracy@25</td>
        <td>accuracy@50</td>
        <td>Время, с (n=5)</td>
        <td>Время, с (n=25)</td>
        <td>Время, с (n=50) </td>
    </tr>
    <tr>
        <td>CountVectorizer + NearestNeighbors</td>
        <td>46,078</td>
        <td>85,621</td>
        <td>90,85</td>
        <td>62,369</td>
        <td>61,886</td>
        <td>62,99899 </td>
    </tr>
    <tr>
        <td>CountVectorizer + PCA(1024) + NearestNeighbors</td>
        <td>41,83</td>
        <td>88,562</td>
        <td>92,81</td>
        <td>0,223</td>
        <td>0,225</td>
        <td>0,208 </td>
    </tr>
    <tr>
        <td>TfidfVectorizer + NearestNeighbors</td>
        <td>46,405</td>
        <td>91,503</td>
        <td>98,693</td>
        <td>62,05017</td>
        <td>63,7106</td>
        <td>62,01344 </td>
    </tr>
    <tr>
        <td>TfidfVectorizer + PCA(512) + NearestNeighbors</td>
        <td>46,732</td>
        <td>85,948</td>
        <td>92,484</td>
        <td>0,223</td>
        <td>0,225</td>
        <td>0,208 </td>
    </tr>
    <tr>
        <td>TfidfVectorizer + PCA(1024) + NearestNeighbors</td>
        <td>46,078</td>
        <td>85,621</td>
        <td>92,157</td>
        <td>0,44</td>
        <td>0,422</td>
        <td>0,44 </td>
    </tr>
    <tr>
        <td>TfidfVectorizer + PCA(2048) + NearestNeighbors</td>
        <td>46,078</td>
        <td>90,196</td>
        <td>97,386</td>
        <td>0,937</td>
        <td>0,942</td>
        <td>0,927 </td>
    </tr>
    <tr>
        <td> Faiss(IVFPQ)</td>
        <td>44,771</td>
        <td>82,026</td>
        <td>89,542</td>
        <td>1,86755</td>
        <td>1,533</td>
        <td>1,599 </td>
    </tr>
    <tr>
        <td>PCA(512) +  Faiss(IVFPQ)</td>
        <td>46,405</td>
        <td>84,641</td>
        <td>89,869</td>
        <td>0,005</td>
        <td>0,006</td>
        <td>0,006 </td>
    </tr>
    <tr>
        <td>PCA(1024) +  Faiss(IVFPQ)</td>
        <td>42,484</td>
        <td>86,275</td>
        <td>90,85</td>
        <td>0,008999</td>
        <td>0,009001</td>
        <td>0,008 </td>
    </tr>
    <tr>
        <td>PCA(2048) +  Faiss(IVFPQ)</td>
        <td>46,405</td>
        <td>84,967</td>
        <td>90,85</td>
        <td>0,020998</td>
        <td>0,017999</td>
        <td>0,014 </td>
    </tr>
    <tr>
        <td>Faiss(IndexIVFFlat)</td>
        <td>45,098</td>
        <td>87,908</td>
        <td>95,098</td>
        <td>9,099564</td>
        <td>9,483988</td>
        <td>9,907769 </td>
    </tr>
    <tr>
        <td>PCA(2048) + Faiss(IndexIVFFlat)</td>
        <td>46,405</td>
        <td>87,908</td>
        <td>94,771</td>
        <td>0,206999</td>
        <td>0,226941</td>
        <td>0,189997 </td>
    </tr>
    <tr>
        <td>PCA(512) + Faiss(IndexIMIFlat)</td>
        <td>46,078</td>
        <td>83,333</td>
        <td>89,216</td>
        <td>0,021999</td>
        <td>0,019998</td>
        <td>0,024 </td>
    </tr>
    <tr>
        <td>PCA(2048) + Faiss(IndexIMIFlat)</td>
        <td>46,405</td>
        <td>90,196</td>
        <td>97,386</td>
        <td>0,588001</td>
        <td>0,546999</td>
        <td>0,541 </td>
    </tr>
    <tr>
        <td>SBERT + Faiss(IndexIMIFlat)</td>
        <td>48,039</td>
        <td>93,137</td>
        <td>96,405</td>
        <td>0,011999</td>
        <td>0,008004</td>
        <td>0,007999 </td>
    </tr>

</table>

## Ранжирование

Модель поиска ближайших соседей (кластеризации) возвращает n кандидатов для каждого интересующего нас трека. 
То есть для 100 треков будет возвращено 5000 (при n=50). Над полученным датасетом можно проводить различные манипуляции, в соответствии 
с запросом пользователя. Например:

	1) Отсортировать по жанрам  
	2) Вывести только каверы  
	3) Вывести только оригиналы  
	4) Вывести наиболее похожие по тексту треки  
	5) Вывести отранжированные к оригиналу треки

Первой задачей заказчика является определение, является ли трек кавером или нет. Поэтому 
к полученному набору треков можно применить модель классификации, или отранжировать новую модель с учетом новых 
признаков внутри групп. В нутри группы можно построить матрицу схожести между текстами. Это может быть хорошим дополнительным признаком.

В качестве классификатора мы возьмем модель CatBoost: 

![alt text](https://github.com/raphael12/YandexMusic/blob/main/images/%D0%A0%D0%B8%D1%81%2013.png)  

![alt text](https://github.com/raphael12/YandexMusic/blob/main/images/%D0%A0%D0%B8%D1%81%2012.png) 

Точность модели на тестовой выборке составляет около 90%. Это значит, что на отобранных данных, возможно проклассифицировать по COVER/ORIGINAL с точностью 90%.


## Выводы 

Перед нами стояла задача разработать метод, для определения оригинальных треков для каверов. Для этого была реализована двустадийная модель, которая на первом этапе отбирала похожие по тексту треки, а на втором этапе применялась бинарная классификация по меткам COVER/ORIGINAL.   

Для построения наилучшей модели класстеризации и поиска ближайших соседей исследовался стек:

        1)CountVectorizer  - Векторизация текста  
        2)TfidfVectorizer  - Векторизация текста  
        3)PCA  -Уменьшение размерности   
        4)NearestNeighbors  - Прямой поиск    
        5)Faiss  - Семантический поиск    
        6)SBERT- Векторизация текста  
        
Результаты представлены в таблице выше  

Видно, что наилучшие результаты по точности и времени работы показывают две связки:

        1)SBERT + Faiss(IndexIMIFlat) с точностью поиска = 	96,405, при n=50 и временем = 0,007999 секунды  
        2)TfidfVectorizer  + PCA(2048) + Faiss(IndexIMIFlat) = 97,386, при n=50 и временем = 0,541 секунды  

У первого варианта скорость поиска очень большая, так как SBERT выдает очень маленькую матрицу после векторизации, но он не будет работать с новыми данными, на которых он не обучался (нет метода transform). Данного недостатка нет у второго подхода. Точность у него чуть выше, но скорость поиска уступает на два порядка.  
Для применения на втором этапе первый вариант необходимо доработать, поэтому мы тестировали финальную модель только со вторым подходом. 
