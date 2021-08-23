# SPD-Ukraine test task

## Задание:
Решить задачу классификации текста, описать процесс решение задачи, подкрепить графиками, описать дальнейшие возможности по улучшению. Максимальное отведенное время 5 рабочих дней. 
В задаче не уточняется целевая метрика, предполагаемая нагрузка, требования к оборудованию.

[Дополнительная информация о задаче](https://github.com/IBerdinskikh/spd-ukraine-tt/blob/main/task/PBLabs-DescriptionClassificationProject.pdf), [Dataset](https://github.com/IBerdinskikh/spd-ukraine-tt/blob/main/dataset/aboutlabeled.csv).

## Решение:
На решение задачи было потрачено 32 рабочих часа, 8 часов я решил оставить на случай критических замечаний и необходимости доработки. 

### 01 Baseline:
1. По ряду признаков данные в датасете выглядят странно. У первого класса очень узкое распределение это синтетические или уже аугментированные данные. Есть сильный дисбаланс между 0 и 1 классом, это нормально.
2. В качестве целевой метрики я выбрал f1, ROC-AUC – выглядит очень оптимистично при сильном перекосе precision и recall.
3. Разбил выборку на train и test. Тестовая выборка используется только в самом конце для финальной проверки модели.
4. Удалил только полные дубликаты из тренировочной выборки. 
5. Разработал уникальный алгоритм, для улучшения распределения данных.
С помощью расстояния Левенштейна рассчитал уникальность наблюдения по отношению к другим наблюдениям (Distance). 
Посчитал веса для каждого наблюдения: 
df['w'] = 1. - df['Distance']
df['w'] = (df['w'] - df['w'].min()) / (df['w'].max() - df['w'].min())
Логика в том, что для модели более значимы уникальные тексты. Мало уникальные тексты оказывают меньше влияния на модель. Перерасчет желательно производить в цикле кросс валидации.
6. Так как данные предназначены для финансовой аналитики, для кросс валидации я хотел использовать RepeatedStratifiedKFold(n_splits=5, n_repeats=20) но ресурсов хватило только на n_repeats=2
7. В качестве baseline модели был выбран spacy + TfidfVectorizer + RandomForestClassifier, ввиду простоты и низкой склонности к переобучению.
8. Для подбора гиперпараметров я использовал optuna (TPESampler)
9. На тестовых данных baseline набрал f1 = 0.516, ROC-AUC = 0.773

[Подробнее](https://github.com/IBerdinskikh/spd-ukraine-tt/blob/main/01%20Baseline.ipynb)

### 02 Fine tuning BERT (продолжение Baseline)
1. Добавил к выходному вектору BERT классификатор. Для уменьшения протечек, стандартизации и удобства работы других разработчиков я реализовал класс классификатора (AboutClassifier) в стандарте sklearn. 
2. Попробовал аугментацию с помощью перевода на другой язык и обратно. Это вызвало смешение и оверфиттинг.
3. Разработал уникальный алгоритм, предназначен для генерации большого числа наблюдений, соответствующих нашему распределению + детекция выбросов.
Логика в следующем, добавляем любым удобным способом (translate, BERT mask, …) в тренировочную выборку аугментированные наблюдения и составляем маску aug_mask. Находим аномалии во всей тренировочной выборке, помечаем как outliers_mask. Передаем полученные маски в cross_val_score. В цикле кросс валидации обучение производим на всех данных кроме выбросов [~outliers_mask], валидацию на всех реальных данных [~aug_mask]. С помощью данного алгоритма, улучшил соотношение классов с 0.879569/0.120431 до 0.560555/ 0.439445 в тренеровачных данных. 
4. На тестовых данных BERT набрал f1 = 0.550, ROC-AUC = 0.852
5. В виду высокой вычислительной сложности я решил отложить дальнейшую работу в направлении BERT.

[Подробнее](https://github.com/IBerdinskikh/spd-ukraine-tt/blob/main/02%20Fine_Tuning_BERT.ipynb)


### Дальнейшие возможности по улучшению
1. Feature engineering, feature selection
2. Добавить multi head в “02 Fine tuning BERT”. Чтобы использовать признаки, полученные в результате feature engineering.
3. Попробовать в качестве следующей модели XGBoost.

Направлений для дальнейших работ очень много, реализованы только базовые шаги.

## Развертывание модели
Подготовка модели к развертыванию, замеры скорости/качества/стабильности модели не проводились в связи с нехваткой времени. Возможна реализация любых вариантов удобных для заказчика.
1. Обычно модель работает через flask веб сервер, отвечая на API запросы.
2. CI/CD – да.
3. Test / prod.
