# Предварительная оценка данных и их коррекция

Здесь я не буду приводить сами данные, поскольку в письме была просьба не публиковать данные залдания.

Далее буду фиксировать все шаги и варианты решения проблем.

Для оценки данных нужно провести несколько исследований VCF файла, поскольку я не знаю есть ли в нем ошибки.

При качественном файле VCF можно сразу начинать GWAS исследование, но в данном случае нужно такой уверенности нет и поэтому я проведу прдеварительное изучения исходных данных.

#### 1. Нужно проверить структуру файла VCF, поскольку он имеет регламентированный формат

команда 
`vcf-validator test_data.vcf`
помогает обнаружить различные ошибки в файле формата VCF

в результате ее работы было обнаружено, что в файле присутствует большое количество строк с пустой колонкой референсного аллеля, что при валидации выглядело примерно так: 

`The column REF is empty at 3:30258913.`
`3:30258913 .. Expected combination of A,C,G,T,N for REF, got [], the offending chars were []`

Если быть точным, то в файле присутствует 7817 строк без референсного аллеля,  посчитать это число можно командой `grep -nE '^[^#].*\t\t' test_data.vcf | wc -l`)

Поскольку референсный геном не указан и у нас нет достаточной информации о нем, то я решил удалить эти строки

Валидатор также вывел информацию об отсутствии версии референсного генома, что не критично для проведения дальнейшего исследования. 
`The header tag 'reference' not present. (Not required but highly recommended.)`
Но это, кстати, исключает возможность использовать референс для исправления предыдущей ошибки. При наличии информации о нем, мы могли бы из референсного генома взять соответствующие позиции и дополнить ими каждую из некорректных строк.

Еще один тип ошибок в этом файле был связан с тем, что в поле альтернативного аллеля ALT присутсвовали нуклеотиды, описанные в поле референсного аллеля REF, что недопустимо по регламенту VCF. 
`3:34514361 .. REF allele listed in the ALT field??`

Чтобы избавиться от этих ошибок необходимо удалить строки без референсного аллеля и строки, где содержатся одинаковые альтернативный и референсный аллели. 

Решение первой проблемы я провел с использованием команды grep, т.к. vcf является обычным текстовым файлом:
`grep -vE '^[^#].*\t\t' test_data.vcf > corrected.vcf`
	ключ -v исключает найденные строки из вывода
	ключ -E использован для возможности применить расширенное регулярное выражение, чтобы избежать экранирования символов. В зтой команде основным элементом поиска стали две табуляции подряд, что говорит о наличии пустой колонки в данной строке. 

Далее для исправления последней найденной мной ошибки (REF allele listed in the ALT field в этом файле я использовал команду norm  программы bcftools.
В найденных утилитой vcf-validator позициях (например, 3:34514361) в колонке ALT содержится аллель, совпадающий с REF. Например, если REF=A и ALT=A,C, то использовать A в ALT недопустимо, поскольку это может привести к путанице при интерпретации вариантов и ошибкам в  дальнейшем анализе.

Инструмент bcftools norm может помочь удалить дублирующиеся аллели в поле ALT, если они совпадают с REF. Далее команда выполняет пайплайн из 2х команд:
`bcftools norm -m - corrected1.vcf | bcftools view -e 'ALT=REF' -o filtered.vcf`
Здесь первая часть разделяет мультивариантные записи в альтернативной аллели на отдельные строки, а вторая часть получает результат от первой и удаляет строки, где ALT=REF, а затем записывает результат в файл filtered.vcf
Возможно есть и более эффективный метод для такой манипуляции.

В результате, в этом файле нет технических ошибок (за исключением отсутствия записи о референсном геноме).

Также нужно проверить, чтобы образцы в файле с фенотипами и в VCF файле совпали. В нашем случае в файле с фенотипами 99 образцов, в VCF - 153 образца. Для дальнейшей работы нужно найти пересечение этих двух наборов и понять сколько образцов можно будет использовать в исследовании. 

Создаю списки образцов в двух файлах:

`cut -f1 test_data.tsv > pheno_samples.txt`
Беру первый столбец из файла test_data.tsv и записваю его в файл pheno_samples.txt. 
В этом файле `wc -l pheno_samples.txt` 99 образцов

`bcftools query -l filtered.vcf > vcf_samples.txt`
С помощью запроса с ключом -l из отфильтрованного VCF файла получаю из него список образцов и записываю их в файл vcf_samples.txt
В этом файле `wc -l vcf_samples.txt` 153 образца

Понятно что эти списки отличаются и их нужно объединить, взяв только то, что присутствует в обоих списках. 
Использую для этого команду `comm` с ключом `-12` и записываю результат в отдельный файл.
`comm -12 <(sort vcf_samples.txt) <(sort pheno_samples.txt) > common_samples.txt`
В файле оказалось 99 идентификаторов

Таким же образом можно найти идентификаторы, которые есть в VCF, но нет в phenotype.txt 
`comm -23 <(sort vcf_samples.txt) <(sort pheno_samples.txt) > vcf_only_samples.txt` 
Файл содержит 54 идентификатора

И можно найти идентификаторы, которые есть в phenotype.txt, но нет в VCF 
`comm -13 <(sort vcf_samples.txt) <(sort pheno_samples.txt) > pheno_only_samples.txt`
Файл не содержит идентификаторов.

В результате все сходится, все 99 образцов есть в файле VCF и теперь можно оставить для исследования 99 образцов. Для этого нужно создать файл с фенотипами, содержащий общие образцы приведу его к формату, совместимому с PLINK (нужны колонки: FID, IID, Phenotype). 
Воспользуюсь awk и проверю пересечение этих файлов.
`awk 'NR==FNR {ids[$1]; next} $1 in ids {print "1\t"$1"\t"$2}' common_samples.txt test_data.tsv > phenotype_common.txt`

Чтобы уменьшить размер VCF файла и упростить дальнейшую работу, можно оставить в нем только общие образцы. Это необязательно, так как PLINK может сам фильтровать образцы позже, но это может сэкономить время при конвертации. И я беспокоюсь о мощности своего местного компа :)

Создаю VCF только с общими образцами
`bcftools view -S common_samples.txt corrected.vcf -o corrected_common.vcf` 

Проверяю количество образцов в новом VCF 
`bcftools query -l corrected_common.vcf | wc -l`

Все в порядке, в VCF осталось 99 образцов.

Далее буду работать с этим файлом.
