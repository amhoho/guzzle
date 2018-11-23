# 0.版本
`2018年11月22日 上午1:36`

# 1. 安装
```
yum -y install curl
composer require guzzlehttp/guzzle
```
# 2. API集合:

## 1. 可选参数的构建Client
```php
require "vendor/autoload.php";
use GuzzleHttp\Pool;
use GuzzleHttp\Client;
use GuzzleHttp\Middleware;
use GuzzleHttp\HandlerStack;
use GuzzleHttp\Psr7\Request;

//构建一个Client
$client = new GuzzleHttp\Client(['base_uri' => 'http://test.com']);

//基于base_uri补足
$response = $client->request('GET', '/test');

//不基于base_uri
$response = $client->request('GET', '/http://baz.com');

//覆盖client请求选项
$response = $client->request('GET', 'http://baz.com',['base_uri等请求选项']);

//不覆盖client请求选项的独立request
use GuzzleHttp\Psr7\Request;
$request = new Request('GET', 'http://foo.com', ['X-Foo' => 'test']);
$client->send($request);

//还可以覆盖request的请求选项
use GuzzleHttp\Psr7\Request;
$request = new Request('GET', 'http://foo.com', ['X-Foo' => 'test']);
$client->send($request, ['headers' => ['X-Foo' => 'overwrite']]);

//请求选项很重要,更多的用法参见https://guzzle-cn.readthedocs.io/zh_CN/latest/request-options.html

//post或put等body传参
['body' => 'foo']
['body' => fopen('http://httpbin.org', 'r')]
['body' => GuzzleHttp\Psr7\stream_for('contents...')]

//cookie
['cookies' =>'your cookie']

//连接的超时等待
['connect_timeout' => 10]

//响应超时
 ['timeout' =>10]

//header
['headers' => [
        'User-Agent' => 'testing/1.0',
        'Accept'     => 'application/json',
        'X-Foo'      => ['Bar', 'Baz']
    ]
]

//代理
['proxy' => 'tcp://localhost:8125']
```

## 2. 重试机制
```php
ini_set("memory_limit", "1024M");//适当的控制内存
use GuzzleHttp\Middleware;
use GuzzleHttp\HandlerStack;
$stack = HandlerStack::create();
$stack->push(Middleware::retry(
    function($retries) { return $retries < 3; },
    function($retries) { return pow(2, $retries - 1); }
));
//$client = new Client....启动Client
```

## 3. 异常处理
```php
use GuzzleHttp\Exception\RequestException;
try {
    $client->request('GET', 'https://github.com/_abc_123_404');
} catch (RequestException $e) {
    echo $e->getRequest();
    if ($e->hasResponse()) {
        echo $e->getResponse();
    }
}
```

## 4. 发送请求
```php
//各种协议形式的请求
$response = $client->get('http://httpbin.org/get');
$response = $client->delete('http://httpbin.org/delete');
$response = $client->head('http://httpbin.org/get');
$response = $client->options('http://httpbin.org/get');
$response = $client->patch('http://httpbin.org/patch');
$response = $client->post('http://httpbin.org/post');
$response = $client->put('http://httpbin.org/put');

//也可以独立使用request的请求
use GuzzleHttp\Psr7\Request;
$request = new Request('PUT', 'http://httpbin.org/put');
$response = $client->send($request, ['timeout' => 2]);

//异步请求,只需要在上面的方法名加上Async
$promise = $client->getAsync('http://httpbin.org/get');
$promise = $client->deleteAsync('http://httpbin.org/delete');
$promise = $client->headAsync('http://httpbin.org/get');
$promise = $client->optionsAsync('http://httpbin.org/get');
$promise = $client->patchAsync('http://httpbin.org/patch');
$promise = $client->postAsync('http://httpbin.org/post');
$promise = $client->putAsync('http://httpbin.org/put');
//异步请求并回调例子1:
use GuzzleHttp\Psr7\Request;
$request = new Request('PUT', 'http://httpbin.org/put');
$promise = $client->sendAsync($request, ['timeout' => 2]);

//异步请求并回调例子2:
use Psr\Http\Message\ResponseInterface;
use GuzzleHttp\Exception\RequestException;
$promise = $client->requestAsync('GET', 'http://httpbin.org/get');

//异步回调处理
$promise->then(
    function (ResponseInterface $res) {
        echo $res->getStatusCode() . "\n";
    },
    function (RequestException $e) {
        echo $e->getMessage() . "\n";
        echo $e->getRequest()->getMethod();
    }
);

//相当于js的Promise.all,异步并发请求例子1:
use GuzzleHttp\Client;
use GuzzleHttp\Promise;
$client = new Client(['base_uri' => 'http://httpbin.org/']);
$promises = [
    'image' => $client->getAsync('/image'),
    'png'   => $client->getAsync('/image/png'),
    'jpeg'  => $client->getAsync('/image/jpeg'),
    'webp'  => $client->getAsync('/image/webp')
];
$results = Promise\unwrap($promises);
echo $results['image']->getHeader('Content-Length');
echo $results['png']->getHeader('Content-Length');

//相当于js的Promise.all,异步并发请求例子2:
use GuzzleHttp\Pool;
use GuzzleHttp\Client;
use GuzzleHttp\Psr7\Request;
$client = new Client();
$concurrency=100;//并发数
//批量生成请求
$requests = function ($total) {
    $uri = 'http://127.0.0.1:8126/guzzle-server/perf';
    for ($i = 0; $i < $total; $i++) {
        yield new Request('GET', $uri);
    }
};
$pool = new Pool($client, $requests(100), [
    "concurrency" => $concurrency,
    'fulfilled' => function ($response, $index) {
        //成功回调数组
    },
    'rejected' => function ($reason, $index) {
//失败回调的数组
    },
]);
$promise = $pool->promise();
$promise->wait();
```

## 5. 响应头处理
```php
$code = $response->getStatusCode(); // 200
$reason = $response->getReasonPhrase(); // OK
echo $response->getHeader('Content-Length');//获得回应内容
//回应中header是否存在
if ($response->hasHeader('Content-Length')){echo "It exists";}
//遍历所有响应头
foreach ($response->getHeaders() as $name => $values) {
echo $name . ': ' . implode(', ', $values) . "\r\n";
}
$body = $response->getBody();//响应主体
$tenBytes = $body->read(10);//读取响应的指定字节
$remainingBytes = $body->getContents();//响应正文
```

## 6.登录,代理,cookie,UA,重定向,传参,SSL,上传等均通过请求选项完成.在第1点有地址











