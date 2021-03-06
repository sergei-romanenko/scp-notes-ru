# Комбинаторные парсеры. Часть 1

**Илья Ключников**

**30 апреля 2010 г.**

В сообщении [Проблемно-ориентированные языки и суперкомпиляция][ho-dsl]
был приведен пример реализации парсеров регулярных выражений через
комбинаторы. Позволим себе взять подмножество языка регулярных выражений
и немного поэкспериментировать с их реализацией через комбинаторные парсеры.

Возьмем простой язык `L`, алфавит которого состоит всего из двух символов -
`A` и `B`. Рассмотрим следующее подмножество регулярных выражений для
этого языка `L`:

    re = a | b | concat re1 re2 | or re1 re2;

* Слово `w` соответствует регулярному выражению `a`, если слово начинается
  с `A`, то есть `w = A w1`.
* Слово `w` соответствует регулярному выражению `b`, если слово начинается
  с `B`, то есть `w = B w1`.
* Слово `w` соответствует регулярному выражению `(concat re1 re2)`, если
  слово `w` представимо в виде `w = w1 w2 w3`, где `w1` - соответсвует `re1`,
  а `w2` - соответсвует `re2`.
* Слово `w` соответствует регулярному выражению `(or re1 re2)`, если оно
  соответствует либо выражению `re1` либо выражению `re2`.

Для удобства чтения `(or re1 re2)` можно записывать как `(re1|re2)`, а
конкатенацию `(concat re1 re2)` - как `(re1)(re2)`.

Результат сопоставление слова языка `L` с регулярным
выражением `re` определяется так: если существует начальная часть слова,
удовлетворяющая выражению, то от слова откусывается эта часть и
возвращается оставшаяся часть слова; если же никакая начальная часть не
соответствует выражению, то возвращается неудача.

Напишем парсеры и соответствующие комбинаторы для языка `L` в духе примера
из [упомянутого послания][ho-dsl], но с одним отличием - в том примере алфавит
языка и представление слов были жестко связаны вместе:

    data Input = Eof | A Input | B Input | C Input;

Мы же разделим понятие алфавит и слова. Конкретный алфавит языка будет
представляться конкретным перечислением букв, а слово языка - это список
букв.

Получаем что-то в таком духе:

    data Symbol = A | B;
    data List a = Nil | Cons a (List a);
    data Option a = Some a | None;

    par1 = concat (or a (concat a b)) (or (concat b b) a);

    or = \p1 p2 word -> case p1 word of {
      Some word1 -> Some word1;
      None -> p2 word;
    };

    concat = \p1 p2 word -> case p1 word of {
      Some word1 -> p2 word1;
      None -> None;
    };

    a = \word -> case word of {
      Nil -> None;
      Cons sym word1 -> case sym of {
        A -> Some word1;
        B -> None;
      };
    };

    b = \word -> case word of {
      Nil -> None;
      Cons sym word1 -> case sym of {
        A -> None;
        B -> Some word1;
      };
    };

Что делают функции - понятно:

* `a` (парсер) - соответствует регулярному выражению `a` - откусывает
  (если можно) от слова символ `A` и возвращает оставшуюся часть. Если
  слово пустое или не начинается с `A`, возвращает неуспех.
* `b` (парсер) - аналогично `a`.
* `or` (комбинатор парсеров) - берет два парсера `p1` и `p2` и делает новый
  парсер, который возвращает результат работы первого парсера, если
  это успех, а в противном случае возвращает результат второго парсера
* `concat` (комбинатор парсеров) - конкатенирует (соединяет) два
  парсера, - делает новый парсер, который возвращает успех, тогда и
  только тогда, когда первый парсер завершается успешно и второй
  парсер, примененный к остатку слова, выплюнутому первым парсером,
  также завершатся успехом.
* `par1` - парсер, соответствующий регулярному выражению `(a|ab)(bb|a)`.

Попробуем применить парсер `par1` к слову `ABB`. `A` - соответсвует
выражению `a`, `BB` - выражению `bb`. Ожидается, что результатом работы
будет `Some w1`, где `w1` - пустое слово.

Поскольку суперкомпилятор [HOSC][] можно применять и в качестве
интерпретатора, [попросим HOSC вычислить выражение][np1]
`par1 (Cons A (Cons B (Cons B Nil)))`,
- как и ожидалось в результате получается `(Some Nil)`.

Пойдем дальше - применим парсер `par1` к слову `ABA`. `AB` - соответсвует
выражению `ab`, `A` - выражению `a`. Опять ожидаем, что результатом работы
будет `Some w1`, где `w1` - пустое слово.

[Вычисляем выражение][np2] `par1 (Cons A (Cons B (Cons A Nil)))`.
Получаем ... `None`!

Как же так? Дерево вычислений, показанное HOSC'ом помогает найти ответ
на вопрос - после некоторого числа шагов вычисление исходного выражения
сводится (редуцируется) к вычислению выражения
`(or (concat b b) a) (Cons B (Cons A Nil))`, которое завершается неудачей.

Можно еще отсуперкомпилировать выражение `par1 w` для неизвестного
слова `w`. [Получается следующий результат][np3]:

    case  w  of {
      Nil -> None;
      Cons u10 x11 ->
        case  u10  of {
          A ->
            case  x11  of {
              Nil  -> None;
              Cons p3 u13 ->
                case  p3  of {
                  A -> (Some u13);
                  B -> case u13 of {
                    Nil  -> None;
                    Cons z10 u8 -> case z10 of {
                      A -> None;
                      B -> (Some u8);
                    };
                  };
                };
            };
          B  -> None;
        };
    }

Видно, что на самом деле мы закодировали регулярное выражение `a(a|bb)`,
или (что то же самое) `(aa|abb)`. Как же так получилось?

Дело в том, что парсер `(or (a) (concat a b))` никогда не будет откусывать
от слова `AB` - из-за того, что если парсер `(concat a b)` способен для
данного слова выдать что-то наружу, то на это способен и парсер `a`, а он
вызывается в нашей реализации первым - и получается, что наш комбинатор
`or` работает для парсера `(or p1 p2)` правильно только в случае,
если `p1` и `p2` не могут выдавать успех на одном и том же слове. Или
(говоря более научно), когда языки, определяемые парсерами `p1` и `p2` не
пересекаются.

Как же решить возникшую проблему? Как сделать, чтобы результирующий
парсер (сконструированный из комбинаторов) пробовал все варианты? В
нашем случае мы сделали так, что парсер возвращает ровно одно успешное
сопоставление. Одно из решений проблемы -- заставить парсер возвращает
не одно единственное сопоставление в случае успеха, а все возможные.
Ясно, что тогда усложняется логика работы комбинаторов парсеров -- они
должны анализировать все сопоставления парсеров, переданных как
аргументы. Таким образом парсер `or p1 p2` должен возращать объединение
результатов парсеров `p1 и p2`. А парсер `(concat p1 p2)` должен к каждому
из вариантов сопоставления `p1` попробовать применить `p2` и тоже объединить
результаты. Этот подход изложен в классической статье Филиппа
Уодлера ["How to Replace Failure by a List of Successes (A method for
exception handling, backtracking, and pattern matching in lazy
functional languages)"][Wadler1985].

Но можно пойти и другим путем - можно каждому парсеру в качестве
дополнительного аргумента передавать парсер, который будет вызван на
следующем шаге сопоставления. Такой подход по духу очень близок стилю
программированию в продолжениях ([Continuation passing style][wiki-cps]).
Действительно, в нашей первой наивной реализации парсер
`(or (a) (concat a b))` действовал по принципу "После нас хоть потоп" -
возвращал всегда первое сопоставление - его не интересовало, пригоден ли
его ответ для обработки следующим парсером в цепочке.

Легко кодируются в таком стиле парсеры для выражений `a` и `b`.

    a = \next word -> case word of {
      Nil -> None;
      Cons sym word1 -> case sym of {
        A -> next word1;
        B -> None;
      };
    };

    b = \next word -> case word of {
      Nil -> None;
      Cons sym word1 -> case sym of {
        A -> None;
        B -> next word1;
      };
    };

Дополнительный аргумент `next` - это следующий парсер. Логика
парсера `a` проста - если слово начинается с `A`, то на остаток слова
натравливается парсер `next`, в противном случае - неудача.

Комбинатор `or` записывается так:

    or = \p1 p2 next word -> case p1 next word of {
      Some word1 -> Some word1;
      None -> p2 next word;
    };

А комбинатор `concat` записывается очень изящно:

    concat = \p1 p2 next word -> p1 (p2 next) word;

Композиция парсеров представляется в виде композиции функций!

Неожиданно возникает вопрос а как применить парсер к слову? Ведь теперь
любой парсер требует своего продолжения! Для этого потребуется
специальный парсер return, который разрывает порочный круг из
бесконечных продолжений:

    return = \w -> Some w;

В итоге для сопоставления выражения `(a|ab)(bb|a)` со
словом `ABA` получается следующая программа:

    data Symbol = A | B;
    data List a = Nil | Cons a (List a);
    data Option a = Some a | None;

    par1 (Cons A (Cons B (Cons A Nil)))

    where

    par1 = concat (or a (concat a b)) (or (concat b b) a) return;

    return = \w -> Some w;

    or = \p1 p2 next word -> case p1 next word of {
      Some word1 -> Some word1;
      None -> p2 next word;
    };

    concat = \p1 p2 next word -> p1 (p2 next) word;

    a = \next word -> case word of {
      Nil -> None;
      Cons sym word1 -> case sym of {
        A -> next word1;
        B -> None;
      };
    };

    b = \next word -> case word of {
      Nil -> None;
      Cons sym word1 -> case sym of {
        A -> None;
        B -> next word1;
      };
    };

[Вычисляем это выражение с помощью суперкомпилятора][sp2] и
получаем `(Some Nil)`! Если посмотреть на трассировку вычислений, то
видно, что новый парсер по очереди перебирает все варианты, пока не
найдет успешный, - грубо говоря, неявным образом составляются все
возможные комбинации парсеров и по очереди применяются ко входному
слову. Первая комбинация завершиласть неудачно - пробуется вторая и т.д.
Можно еще сказать, что новый комбинатор `or` как бы выносит разбор
вариантов из глубины регулярного выражения наружу. То есть в каком то
смысле переписывает регулярное выражение:

    (re1|re2)re3 -> ((re1 re2) | (re1 re3))

[Вычисление][sp1] `par1 (Cons A (Cons B (Cons B Nil)))` тоже выдает
`(Some Nil)`.

Платой за красоту такой записи регулярных выражений с помощью
комбинаторов являются накладные расходы интерпретатора во время
вычислений. Ведь выражение `(a|ab)(bb|a)` неявно трансформируется так:

    (a|ab)(bb|a) -> (a(bb|a))| (ab(bb|a)) -> abb|aa|abbb|aba

И наш парсер по очереди перебирает эти 4 варианта - то есть в худшем
случае он 4 раза перемещается по входному слову туда-сюда. При более
сложных выражениях парсер может бегать по входному слову туда-сюда очень
много раз. Это особенно проявляется в случаях, когда входное слово не
соответствует регулярному выражению, - прежде чем выдать неудачу парсер
попробует все варианты.

Существует отдельная область математики, изучающая только регулярные
выражения. Доказано, что любое (классическое) регулярное выражение можно
превратить в детерминированный  конечный автомат, который кушает по
одному символа входного слова и выдает результат (успех или неудачу) за
минимальное число шагов (поеданий очередного символа). Кодировать же
регулярные выражения в виде автоматов для программиста задача
неблагодарная и ведущая к потенциальным ошибкам. Поэтому многие языки (и
библиотеки) поддерживают регулярные выражения удобным для программистов
способом - а внутри они преобразуют красивую и изящную запись
регулярного выражения в потенциально большой, но быстрый конечный автомат.

А попробуем сопоставить наше игрушечное регулярное выражение с
произвольным словом `w`, то есть вычислить

    concat (or a (concat a b)) (or (concat b b) a) return w

с помощью суперкомпилятора [HOSC][]. [Результат такой][sp3]:

    case  w  of {
      Nil -> None;
      Cons p40 v20 ->
        case  p40  of {
          A ->
            case  v20  of {
              Nil -> None;
              Cons v1 u30 ->
                case  v1  of {
                  A -> (Some u30);
                  B ->
                    case  u30  of {
                      Nil -> None;
                      Cons y18 p22 -> case  y18  of {
                        A -> (Some p22);
                        B -> (Some p22);
                      };
                    };
                };
            };
          B -> None;
        };
    }

Видно, что отсуперкомпилированный парсер не бегает по слову туда-сюда, а
продвигается по слову только вперед.

## Заключение

Было показано, как с использованием комбинаторного подхода реализуется
проблемно-ориентированный язык для работы с подмножеством регулярных
выражений. Реализовать этот язык у нас получилось только со второго
раза, что является подтверждением того, что хотя работать с
комбинаторным DSL легко и приятно, реализовать даже простой DSL не так
то уж и тривиально. Оказалось, что суперкомпилятор [HOSC][] способен
устранять накладные расходы, вызванные использованием изящного, но
требовательного к ресурсам DSL, реализованный через комбинаторы.

Мы реализовали комбинаторный DSL только для подмножества языка
регулярных выражений. В следующий раз мы расширим наш маленький DSL до
полной поддержки классических регулярных выражений и посмотрим какие
проблемы при этом могут возникнуть.

---

[Оригинал послания и комментарии](http://metacomputation-ru.blogspot.ru/2010/04/1.html)

[ho-dsl]: 10-ho-dsl.md

[HOSC]: https://sergei-romanenko.github.io/hosc-docs/

[Wadler1985]: https://link.springer.com/chapter/10.1007/3-540-15975-4_33

[wiki-cps]: http://en.wikipedia.org/wiki/Continuation-passing_style>

[np1]: http://hosc.appspot.com/view?key=agpzfmhvc2MtaHJkcjULEgZBdXRob3IiGmlseWEua2x5dWNobmlrb3ZAZ21haWwuY29tDAsSB1Byb2dyYW0YuZECDA

[np2]: http://hosc.appspot.com/view?key=agpzfmhvc2MtaHJkcjULEgZBdXRob3IiGmlseWEua2x5dWNobmlrb3ZAZ21haWwuY29tDAsSB1Byb2dyYW0YiaECDA

[np3]: http://hosc.appspot.com/view?key=agpzfmhvc2MtaHJkcjULEgZBdXRob3IiGmlseWEua2x5dWNobmlrb3ZAZ21haWwuY29tDAsSB1Byb2dyYW0Y2bACDA

[sp1]: http://hosc.appspot.com/view?key=agpzfmhvc2MtaHJkcjULEgZBdXRob3IiGmlseWEua2x5dWNobmlrb3ZAZ21haWwuY29tDAsSB1Byb2dyYW0YoZkCDA

[sp2]: http://hosc.appspot.com/view?key=agpzfmhvc2MtaHJkcjULEgZBdXRob3IiGmlseWEua2x5dWNobmlrb3ZAZ21haWwuY29tDAsSB1Byb2dyYW0Y8agCDA

[sp3]: http://hosc.appspot.com/view?key=agpzfmhvc2MtaHJkcjULEgZBdXRob3IiGmlseWEua2x5dWNobmlrb3ZAZ21haWwuY29tDAsSB1Byb2dyYW0YmvIBDA
