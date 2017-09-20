# Опыт внедрения PSR стандартов в одном легаси проекте

Всем привет!
В этой статье я хочу рассказать о своем опыте переезда на “отвечающую современным трендам” платформу в одном legacy проекте.

<cut/>

Все началось примерно год назад, когда меня перекинули в “старый” (для меня новый) отдел.
До этого я работал с Symfony/Laravel. Перейдя на проект с самописным фреймворком количество WTF просто зашкаливало, но со временем все оказалось не так и плохо.
Во-первых, проект работал. Во-вторых, применение шаблонов проектирования просматривалось: был свой контейнер зависимостей, ActiveRecord и QueryBuilder.
Плюс был дополнительный уровень абстракции над контейнером, логгером, работе с очередями и зачатки сервисного слоя(бизнес логика не зависела от HTTP слоя, кое где логика была вынесена из контроллеров).

Далее я опишу те вещи, с которыми трудно было мириться:
#### 1.  Логгер [log4php](https://logging.apache.org/log4php/index.html) #### 
Сам по себе логгер работал и хорошо. Но были жирные минусы:
-  Отсутствие интерфейса
-  сложность конфигурации для задач чуть менее стандартных (например, слать логи уровня error в ElastickSearch).
-  Подавляющее большинство компонентов мира opensource зависят от интерфейса Psr\Log\LoggerInterface. В проекте все равно пришлось держать несколько логгеров.


#### 2-6. Контроллеры были вида: #### 
```php
<?php

class AwesomeController
{
   public function actionUpdateCar($carId)
   {
       $this->checkUserIsAuthenticated();
       if ($carId <= 0) {
           die(trans('Машина не найдена'));
       }
       $car = Car::findById($carId);
       $name = @$_POST['name'];
       container::getCarService()->updateNameCar($car, $name);
       echo json_decode([
           'message' =>'Обновление выполнено'
       ]);
   }
}
```
-  **die **посреди выполнения кода приложения
-  **echo **еще до выхода из контроллера. 
-  Аутентификация пользователей подключалась в каждом контроллере отдельно
-  Классические **$_POST** и **@**.
-  Отсутствовала возможность внедрять сервисы в контроллер.

#### 7 . Отсутствие глобального логирования ошибок приложения #### 
Максимум, что можно было найти в логах - это текст сообщения. Стек трейс ты мог  получить только после воспроизведения бага у себя на машине. Или специально ставя блоки try/catch в каждом экшене контроллера и добавив логирование ошибки.

#### 8 . Конфигурирование контейнера зависимостей #### 
Он напоминал symfony/container времен 2.4. Нужно было обязательно регестрировать каждый сервис и описывать как его собрать. 
Autowiring появился сравнительно недавно, но после Laravel хотелось простоты и удобства. К тому же отсутствие autowiring снижало желание программистов писать отдельные сервисы под отдельную бизнес задачу(Single Responsobility) так как приходилось лезть в конфиг контейнера и 10 минут править его. При этом всегда есть вероятность ошибиться.

#### 9 . Роутинг #### 
Роутинг был логичен и прост, по мотивам в Yii1.
Url вида www.carexchange.ru/awesome_controller/update_car означал роутинг до контроллера AwesomeController и метода actionUpdateCar.
Но, к сожалению, были вложенные поддиректории сайта и приходилось создавать url’ы вида
www.carexchange.ru/awesome_controller_for_car_insite_settings_approve/update_car
Это не напрягает, но ограничение странное

#### 10 . Хардкод url’ов #### 
Роутинг был слишком прост, поэтому отсутствовала возможность генерации url автоматически (зачем усложнять). Это привело к тысячам ссылок, которые были захардкожены и в php и js. Мы, конечно, редко меняем url’ы, но иногда такое случается. И искать их по проекту сложно.


С приходом еще одного программиста стали подниматься вопросы о возможности рефакторинга, было желание сделать “по человечнее”. “По человечнее” - читай привычнее для современного веб-разработчика. Читать и поддерживать существующий код сложно->долго->дорого.

После нескольких обсуждений с руководством был получен зеленый флаг и началась работа над proof of concept.

С самого начала было принято решение придерживаться современных стандартов (Это и рекомендации PSR, и следование стандартному поведению других фреймворков). Новые разработчики работавшие на любом современном фреймворке должны были понять где лежат Middleware, где контроллеры, как собираются сервисы и где писать бизнес логику.

Если посмотреть внимательнее на озвученные претензии, то становится видно: страдает код уровня приложения(т.е. контроллеры) и инфраструктурный слой(контейнер).
Бизнес логика была написана отдельно и не зависела от уровня HTTP - ее оставляем как есть. Active Record и QueryBuilder также не трогаем, т.к. они работали и не сильно отличались от той же doctrine/dbal. 

#### Выбор фреймворка ####
На самом деле выбор тут был не велик. Тащить весь laravel или symfony ради слоя над http нет смысла. А нужные компоненты всегда можно подключить через composer.
Серьезный выбор был между двумя микро-фреймворками: Slim и Zend.
Оба этих фреймворка полностью поддерживают PSR-7 и PSR-11. 

Почему не Lumen? Главная причина конечно же в том, что Lumen сложно назвать “микро” вкупе со всем этим [добром](https://github.com/laravel/lumen-framework/blob/5.5/composer.json#L19-L43). Встроить Lumen в существующий проект сложно. Контейнер зависимостей легко не подменишь (необходимо соблюдение контракта illuminate). Контракт PSR-7 фреймворк поддерживает, но внутренности все равно работают с symfony/http-foundation.

Сначала я всерьез взялся за Zend. Но потратив 2 дня, посмотрев на реализацию приложения в идеологии "все middleware", увидев как формируется конфиг контейнера, я с ужасом представил как буду объяснять нашим junior’ам чем invokables отличается от factories, и когда писать aliases.
В общем Zend хорош, академичен. Приложение работает через pipeline и middleware (Красота! Перфекционисты должны оценить). Но я испугался высокого порога входа. А ведь переезд должен быть легким, в идеале незаметным.
Затем я переключился на Slim. Его внедрение в проект заняло меньше дня. Разруливание роутов (старых и новых контроллеров) было реализовано через middleware. На нем и остановился. В надежде когда-нибудь перейти на pipeline с middleware PSR-15.


#### Выбор контейнера ####
Здесь я просто скажу что остановился на league/container, и попытаюсь объяснить свой выбор.
1. это поддержка PSR-11.
Сейчас большинство контейнеров уже поддерживают PSR-11, но год назад лишь малая часть поддерживала container/interop интерфейс.
2. Автовайринг.
3. Синтасис довольно прост, в противопоставление тому же [zend-servicemanager] (http://zendframework.github.io/zend-expressive/features/container/zend-servicemanager). 
4. Сервис провайдеры, позволяющие писать модули еще более изолированно.
В illuminate/container провайдеры регистрируются в приложении, в league/container провайдеры оказывают влияние только на контейнер. Таким образом приложение не зависит от сервис провайдеров.
5. Делегирование контейнеров. Это ”фича” оказалась решающей для этапа замены контейнера, поэтому раскрою ее подробнее.
При желании внутри league/container может быть несколько PSR-11 совместимых контейнеров.
Возможный сценарий: вы решили сменить ваш старый контейнер на  Symfony/Container. Чтобы переходить постепенно вы можете подключить league/container и в делегаты поместить и ваш старый контейнер и контейнер симфони.
При поиске сервиса ваш старый контейнер будет опрашиваться самым первыми, затем будет поиск в Symfony/Container. 
На следующем этапе вы сможете перенести описания всех сервисов в Symfony/Container и оставить только его. Так как код зависит от PSR-11 интерфейса - изменения минимальны.


#### Выбор абстракции над HTTP ####
Тут всего 3 варианта: 
-  [symfony/psr-http-message-bridge](https://github.com/symfony/psr-http-message-bridge)
-  [zendframework/zend-diactoros](https://github.com/zendframework/zend-diactoros)
-  [Реализация slim](https://github.com/slimphp/Slim/tree/3.x/Slim/Http)

Кстати Slim [движется](https://github.com/slimphp/Slim-Http) к выделению реализации http в отдельный пакет(ожидается в ветке 4.0).
Symfony bridge использовать не хотелось по причине лишнего кода и лишней зависимости. Т.к. Slim ни в чем нас не ограничивает, предпочтение было отдано zend/diactoros. Это только увеличило независимость кода приложения от http слоя.

#### Логированиe ####
Тут ничего кроме monolog в голову не приходит. Его и прикрутили.

#### Роутинг ####
В Slim из коробки есть FastRoute. С его помощью мы ввели именованные роуты. И 
вынесли генерацию URL в глобальный хелпер ([Как здесь](https://github.com/laravel/framework/blob/5.5/src/Illuminate/Foundation/helpers.php#L780))

#### Ну и что изменилось? ####
И сейчас наш контроллер может выглядеть так:
В качестве бонуса в зависимости от окружения сейчас мы можем подменять сервисы и гонять функциональные тесты.

```php
<?php

namespace Controllers;

use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Message\ResponseInterface;
use Zend\Diactoros\Response\JsonResponse;
use Domain\Car\Services\CarService;

class myAwesomeController
{
   /**
    * @var CarService
    */
   private $carService;

   public function __construct(CarService $carService)
   {
       $this->carService = $carService;
   }

   public function actionUpdateNameCar(ServerRequestInterface $request, $carId): ResponseInterface
   {
       if ($carId <= 0) {
           throw new BadRequestException('Машина не найдена');
       }
       $car = $this->carService->getCar($carId);
       $name = $request->getParsedBody()['name'];
       $this->carService->updateNameCar($car, $name);

       return new JsonResponse([
           'message' => 'Do Something выполнено'
       ]);
   }
}
```
#### В заключение ####
Как вы видите, много практик из принципа “так проще” позаимствовано из Laravel. Он действительно задает тренды.

Приложение получило новый фреймворк уже после того, как проработало 7 лет. Насколько я знаю, старый самописный фреймворк также появился не сразу. И никто не даст гарантий, что мы не захотим сменить фреймворк через 5 лет. Поэтому мы постарались максимально абстрагироваться от фреймворка и реализаций.
Теперь контроллеры зависят от PSR-7 совместимых запросов и возвращают PSR-7 ответы. Приложение зависит от PSR-11 контейнера. Slim работает через middleware и добавлять общую логику стало проще (Логирование ошибок, обработка ошибок пользователям). Автовайринг контроллеров прекрасно работает, по сути контроллеры стали сервисами.

Кстати [здесь](https://github.com/PHP-DI/Slim-Bridge/blob/master/src/ControllerInvoker.php#L47) можно подсмотреть пример включения autowiring в slim.

Конечно, наше приложение продолжает развиваться и мы уже перешли на стандартные интерфейсы кэширования и событий, но их внедрение было чуть менее кроваво:).
