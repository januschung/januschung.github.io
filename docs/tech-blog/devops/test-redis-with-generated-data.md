# Test Redis with Generated Data

Recently I need to test out Redis integration with a Ruby on Rails application. I wanted to test how fast Redis is with 1GB of data. 

Instead of manually generating dummy data, I recalled using the Faker library a couple of years ago during my time as a full-stack developer.

## What is Faker?

Faker is a library that generates fake data such as names, addresses, emails, and phone numbers. It’s a great tool to create realistic data without the need for manually making sample datasets.

## Installation

Faker supports different programming languages. I chose Python this time.

```
pip install faker
```

## Code to generate 1GB of simple key value pair data

``` python
from fake import Faker
import csv

fake = Faker()

def get_text():
  return ''.join(fake.text(1000)).replace('\r', '').replace('\n', '')

def get_key_and_text_row():
  return [n, get_text()]

with open('large.csv', 'w') as csvfile:
  write = csv.writer(csvfile)
  for n in range(1, 1000000):
    writer.writerow(get_key_and_text_row(n))
```

## Sample output

```
1,Interest local deep college particularly and what. Movie bit task matter likely tend.Quality sometimes Democrat join weight minute do. Company event indicate director great pay specific. Continue bill all collection.Help training enter everybody amount. Middle western administration page heart leader similar. Threat once up authority voice give.Customer safe write people. Employee sell throw as so strategy.Drug really control call offer. Effect west service this under program ask contain.Although able region new wear suggest glass. Risk station project including.Because kitchen light enough. Reflect require name democratic wrong without. Begin three fire baby develop federal.Source husband turn attack. Leader win floor young. Member change peace population account high cover. Add but stay crime himself.Sell stand time sure cut decade system.Open fire hair.Leg six event ability other. Nature agree serve affect fund act. Long star improve spring individual.
2,Station citizen have piece plant. Look science realize stuff television.Single civil poor television actually must. Parent song capital.Too have grow red. Discussion its record you like impact paper.Actually total those main range town. Bit myself future no institution. Suggest unit own at rock personal.Shoulder always today understand especially ball push. Stay appear sort century pull central seven. Site trip memory his start.Member instead option side if result west. Address audience strategy say receive.Candidate later under item yeah will business. Player dog myself purpose.Look trip production water power whether wait rich. Keep sell low if. Quality Republican action main. Sit all see watch same.Lot continue usually wish size. Than morning common hear particular true. Fish minute about.Firm party prepare shoulder main determine health seat. White remain hold card establish interview.Over thousand job minute various adult. Western bill some. Still animal measure.
3,Church behavior investment protect current every fast reach. Including employee reflect material.Center member relate than war up fear. According discuss pay camera activity home fill soldier.Use table view these pattern popular.Song choose street fish must small. Firm beyond across room other.Ago find traditional listen. Collection learn tell agent newspaper way. Maybe mouth executive country. After future region theory with certainly of.Carry eight spend up.Believe teacher score avoid article. Civil movie eight song across task raise.Plant take painting song. Federal people character animal. Experience fill may.Until someone tree newspaper ball. Rest position spring PM claim. Various animal end parent pick hand great. Help tell hit modern artist involve begin final.Plant certainly father heavy reflect anyone. Parent town make against speak. Agency cover film week plant.Board hope let respond majority sister.
4,Capital behind tree note whose. Loss condition program. Rest ball including game.View recent matter science red reason choice necessary. Magazine style defense guy special on special. Sell wait per under. Four skin fear yard rather yes responsibility director.Job none player serve service action. Teach phone wait country audience. Especially can source practice. Ground Republican student remember.Grow throughout research understand. Marriage head arm. Share current father successful action future politics.Environment group include executive news model poor worker. Know spend during artist between in morning.Camera easy street side executive note question. Human various someone moment.Require national try continue conference. Property notice structure program fall movie.Small senior big teacher southern class. Point leg social black. How see investment maybe indeed grow.Goal open story tell. Bit fill car think.
5,We movement pull with near lead. Rock popular when control carry.Treat similar although. Store student five ok bed choice could. Five best community almost.Career these PM provide him. Attorney economic sell behind under wear Congress international. Worry serious social modern law paper.Figure impact employee wall interview major thank serious. Civil protect shake ask. Red hear leader training establish.Dark I role. Avoid defense cut husband culture.Begin account than practice. Table animal it speak.Laugh quickly before eat. Lose about page more. Home new issue.Successful send race road he. Parent must poor box. With wear run.Rock moment allow left system receive decide company. Seven ten then. Better point produce young these study person. American us authority off yes pay none.Our sit clear similar why impact. Could sense when team conference democratic bed mouth. Some whether certainly ago place. Story child matter price thus.
...
```

## Bring up a local Redis with docker

``` bash
docker run --name my-redis -p 6379:6379 -d redis
```

## Load the csv to docker Redis
``` bash
docker cp large.csv my-redis:/tmp
docker exec -it my-redis -- sh
cat /tmp/large.csv | awk -F',' '{print " SET \""$1"\" \""$2"\" \n"}' | redis-cli --pipe
```

## Configuration in Rails

Setup of Redis is different for different environments. For development, local docker is used. For testing, I use a mock Redis library. For production, I use an Elasticache instance which has password protection. 

Here is the setup:

``` ruby
# config/environments/development.rb
$redis = Redis.new({host: 'localhost', port: 6379, db: 0})

# config/environments/development.rb
$redis = MockRedis.new

# config/environments/development.rb
$redis = Redis.new({host: ENV['REDIS_HOST'], port: ENV['REDIS_PORT'], password: ENV['REDIS_PASSWORD'], ssl:true})
```

## Sample Rails controller and unit test

To keep it simple, I just want to get the value from the cache when the corresponding key is provided.

Here is the sample implementation:


``` ruby
# routes.rb

get '/redis/:id', to: 'redis#index'


# redis_controller.rb

class RedisController < ApplicationController
  
  def index
    value = $redis.get(params[:id])
    if value
      render json: {key: params[:id], value: value}
    else
      render json: {warning: "value not found from the cache"}
    end
  end

end

# redis_controller_test.rb

require "test_helper"
require "mock_redis"

class RedisControllerTest < ActionController::TestCase

  setup do
    $redis.set(1, "whatever")
  end

  test "should return value if exists" do
    get :index, params: {id: 1}, as: :json
    assert_response 200
    assert_equal "{\"key\":\"1\",\"value\":\"whatever\"}", @response.body
  end

  test "should return warning message if value not exist" do
    get :index, params: {id: 2}, as: :json
    assert_response 200
    assert_equal "{\"warning\":\"value not found from the cache\"}", @response.body
  end

end