--------------------------------------------------------------------------------

# Бизнес часть

---

### Проблемы текущей реализации

 - очень медленная обработка транзакций(1 tx/sec)
 - лимиты не работают.(кроме контроля того что у пользователя никогда в ноль не уходит и то это не лимит - но это ограничение в БД)
 - история транзакций - totalDebitAmount & totalCreditAmount

---

### Причины почему медленно обрабатываются

 - Длинные транзакции в базе данных(~100 sql на одну транзакцию в бд - причем это записи и чтение) которые занимают ресурс на все время обработки.
   Это ненормально для высокой конкурентности. А так как у нас есть счета которые везде и постоянно используется - это наш случай.
 - Ресурс в данном случае представляет собой остаток на счете + лимит.
   Что само по себе неправильно, так как это разные сущности и не должны блокировать друг друга потому что это независимые процессы,
   а сейчас это одна сущность(таблица)

---

### Проблемы с лимитами

 - Лимиты по датам смотрят назад в историю вместо того чтобы хранить текущее значение лимита.
   Это приводит к тому что лимиты по датам не гарантируют их соблюдения в общем случае.
   Потому что 2 транзакции могли проходить параллельно и одна еще не успела увидеть другую.
   Плюс из-за этого сам функционал получился сложным в поддержке и скорости поэтому изменения могут вносить проблемы.
 - Нет способа сказать чем заблокированы деньги.
 - Существуют(надеюсь уже нет) счета с непонятно чем заблокированными средствами.
 - Если лимит создан во время транзакции которая его превысит - транзакция не пройдет завершения,
   хотя мы уже взяли обязательство тем что ответили что "средства заблокированы"(редкая ситуация, но может стрельнуть).
   Другой вариант реализации с текущей системой - не валидировать, но тогда выходит что лимит превышен.

---

### История транзакций

 - Любые изменения остатков на счетах должны отображаться в истории транзакций.
   Сейчас это 2 независимых процесса, хотя это всегда должно проходить атомарно.
 - Суммы totalDebitAmount и сумма totalCreditAmount по всем кошелькам не дают 0, причем эта сумма почему-то увеличивается все время
   Хотя сумма всех кошелько и дает 0.

---

### Решения для исправления проблем

 - Разделить на 3 подсистемы: олпей, процессинг(движение средств, соблюдение лимитов, управление лимитами), тарифы(расчет тарифов и управление тарифами)
 - Перенести валидацию лимитов на уровень ограничений в базе данных
 - Использование коротких транзакций(2 операции за транзакцию) при записи максимально минимизируя время блока(а не fetch-doLogic-persist)

---

### Новый процессинг

Лимиты будут на пользователя, а не на категорию. Это значит что теперь если будет добавляться лимит мы
будем добавлять для пользователя такой лимит и увеличивать его по всем тем новым транзакциям которые успели создаться за время создания лимита.
Таким образом создание лимита - асинхронная операция.

---

### Новый процессинг

 - Создания лимитов и лимитных планов - дорогие операции и выполняются не сразу, асинхронно.
 - Лимитный план и лимит может не создастся если существуют лимиты которые уже превышены.
 - Лимиты нельзя редактировать(можно будет допилить потом). Пока можно удалить лимит и создать новый.
 - У пользователя будет не более 1 тарифный план.(либо ни одного и добавляем все нужные лимиты руками)
 - Существует стадия лимита где он еще не считается созданным, но по нему уже проходят проверки и
он может завалить пару транзакций по этому лимиту (операция pending прошли неуспешно). Если лимит создать не удалось он более не активен и проверок по нему более не будет

---

### Новый процессинг. Что не будет сделано

 - Не закладываем масштабирование на много БД(хоть это и возможно) чтобы не затягивать разработку.
Этого должно хватить(640 kB is enough).
 - False-positive лимитов: Параллельная обработка лимитов может привести к тому что 2 транзакции завалится,
хотя при последовательной бы завалилась только одна(очень редкий сценарий, главное что обратный сценарий невозможен)
 - Управление аккаунтами, лимитами и лимитными планами будет в новом АПИ процессинга. Бота сможет потом к этому
прикрутить UI. Всего этого не будет в олпее(кроме аккаунтов, они будут через олпей создаваться, но хранится
все равно в новом модуле)

--------------------------------------------------------------------------------

# Техническая часть(TODO)

[processing.jh file](processing.jh)
[studio](https://start.jhipster.tech/jdl-studio/)

### Создания нового платежа

 - Если во время обработки создания транзакции потерялась связь между Allpay и Processing,
необходим будет повторный вызов со стороны Allpay, даже если платеж в Allpay уже отменен(либо выставлять таймаут).

create {
  1. Создается сущность Payment со статусом
  2. Создаются все дочерние BalanceDelta к пейменту(можно параллельно).
  3. На каждый BalanceDelta создаются все дочерние LimitBalanceDelta в статусе DRAFT(можно параллельно).
     Это делается на основании того какие лимиты привязаны к аккаунту, на который ссылается BalanceDelta.
     Лимиты учитываются только в статусах DRAFT и CREATED.
  4. Из всех созданных LimitBalanceDelta найдем которые совпадают по OperationType и увеличим лимит. То есть
     increase limit(подробнее далее) where limitBalanceDelta.limit.limitType = limitBalanceDelta.balanceDelta.operationType and payment = ...
  5. Если все операции прошли успешно статус Payment'а выставляется PENDING и возвращается ответ "OK", status = текущий статус.
     Начиная с этого момента все последущие запросы будут возвращать статус транзакции не повторяя транзакцию. Можно например так:
     https://stackoverflow.com/questions/6722344/select-or-insert-a-row-in-one-command
     Если хотя бы один лимит не смог увеличится - статус Payment'а выставляется как MARKED_FOR_DECLINATION и возвращается ответ соответственно
}

complete(асинхронный*) {
  1. Находится существующая сущность Payment со статусом PENDING и ее статус меняется на MARKED_FOR_COMPLETION
  2. Возвращается ответ ОК и возвращается соответствующий статус.
}

decline(асинхронный*) {
  1. Находится существующая сущность Payment со статусом PENDING или DRAFT и ее статус меняется на MARKED_FOR_DECLINATION
  2. Возвращается ответ ОК и возвращается соответствующий статус.
}

timer {
  Для всех MARKED_FOR_COMPLETION транзакций:
    1. Дозавершаем шаг create.4 - находим все LimitBalanceDelta в статусе DRAFT и выполним их уменьшив лимиты и переведя в статус APPLIED
    2. Когда все лимиты в статусе APPLIED можно переводить платеж в статус COMPLETED и сгенерировать по всем изменениям средств историю транзакций.

  Для всех MARKED_FOR_DECLINATION транзакций:
    1. Находим все LimitBalanceDelta в статусе APPLIED и выполним обратную операцию их уменьшив лимиты и переведя в статус REVERTED
    2. Когда все лимиты в статусе REVERTED или DRAFT можно переводить платеж в статус DECLINED
}

PS:
асинхронный* - ответ мгновенный, но не гарантирующий завершение операции - завершится позже в таймере

### Добавление нового лимита

1. Создается лимит(параметры = maxValue, operationType, updateStrategy) на аккаунт acc:
    status = DRAFT
    latestPaymentApplied = acc.latestPaymentApplied
    currentValue = считается на основании тех пейментов где payment.paymentNumber < limit.latestPaymentApplied.paymentNumber
   Любой повторный вызов будет возвращать тот же статус.
2. Отвечаем OK - в процессе создания.
   Если сломались потому что уже currentValue < maxValue. Отвечаем fail и не создаем лимит вообще.

timer {
  Находим все лимиты в статусе DRAFT и для каждого {
    1. "Догоняем" по всем лимитам. делая шаги из create метода используя limit.latestPaymentApplied.paymentNumber
    2. Если все лимиты удалось увеличить - переводим лимит в статус CREATED.
       Если какой-то лимит не удалось увеличить - переводим лимит в статус CREATION_FAILED
  }
}

### Создание нового аккаунта

Ничего хитрого:

1. Создается новый аккаунт:
  статус = Activated
  balance = 0
  accountNumber - из параметра
  latestPaymentApplied = 0

### Создание нового тарифного плана

1. По аналогии создается тарифный план в статусе DRAFT

timer {
  На каждого пользователя {
    Создаются лимиты из тарифного плана. Если создать удалось статус плана переводится в CREATED,
    иначе - CREATION_FAILED
  }
}

При удалении лимита из лимитного плана удаляются все лимиты на всех аккаунтах, только потом удаляется сам лимит из плана.

### Реализация лимитов

Лимит представляет собой:
 1. текущее значение
 2. значение выше которого нельзя пойти(ограничение БД)
 3. функция которая принимает остаток на счете и вычисляет значение при пополнении.*
 4. функции изменения лимита(параметр - сумма изменения) при *
   - увеличении баланса(complete - получатель**)
   - уменьшении баланса(complete - отправитель**)
   - интенте увеличения баланса(create - получатель***)
   - интенте уменьшения баланса(create - отправитель***)
 5. статус
 6. запись о том какой последний payment применен к лимиту

<pre>
*Пункты 4 и 5 определяются через LimitUpdateStrategy
**Требуется чтобы такая операция никогда не нарушала лимит. Здесь всегда движение вниз(лимита, но не баланса!), так что лимит не может быть нарушен.
  Причем такое уменьшение лимита не может быть отменено, кроме создания новых корректирующих транзакций.
  Потому что в общем случае это не всегда можно сделать отмену уменьшения так как лимит уже уменьшился другой транзакцией.
  (например кто-то использовал деньги для оплаты чего-либо)
***Здесь всегда движение вверх, так что лимит может быть нарушен(например у пользователя нет денег).
   Это движение может быть отклонено в процессе.
</pre>

Примеры:

Не выше X(сумма хранения, storage limit):
 - Создание { текущее значение лимита = текущее значение на счете }
 - Интент увеличения { текущее значение лимита += сумма увеличения }
 - Интент уменьшения { <пусто> }
 - Увеличение { <пусто, вся работа уже сделана в интенте> }
 - Уменьшение { текущее значение лимита -= сумма уменьшения }

Не ниже нуля(как у клиентов):
 - Создание { текущее значение лимита = текущее значение на счете * -1 }
 - Интент увеличения { <пусто> }
 - Интент уменьшения { текущее значение лимита += сумма увеличения }
 - Увеличение { текущее значение лимита -= сумма уменьшения }
 - Уменьшение { <пусто, вся работа уже сделана в интенте> }

Нельзя тратить более X за период N дней:
 - Создание { текущее значение лимита = сумма всех кредитовых операций за последние N дней начиная с состояния аккаунта(account.latestPaymentApplied) }
 - Интент увеличения { <пусто> }
 - Интент уменьшения { текущее значение лимита += сумма увеличения }
 - Увеличение { <пусто> }
 - Уменьшение { <пусто> }
Далее в таймере раз в сутки:
Находить все увеличения лимитов которые выходят за пределы месяца и делать revert-операцию по ним.

Не более 10 транзакций за период N дней:
 - Создание { текущее значение лимита = количество всех кредитовых и дебетовых операций за последние N дней начиная с состояния аккаунта(account.latestPaymentApplied) }
 - Интент увеличения { текущее значение лимита += 1 }
 - Интент уменьшения { текущее значение лимита += 1 }
 - Увеличение { <пусто> }
 - Уменьшение { <пусто> }
Далее в таймере раз в сутки:
Находить все увеличения лимитов которые выходят за пределы N дней и делать revert-операцию по ним.
