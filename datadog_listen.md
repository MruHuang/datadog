[hackmd](https://hackmd.io/@pONMMCWcRMSASTBgLGLkyg/mru_dadadog_listen)

dadadog : 塞入監控程式
===================

Created by mru huang, last modified on Apr 13, 2020

datadog塞入監控程式
-------------

[https://docs.datadoghq.com/tracing/manual\_instrumentation/php](https://docs.datadoghq.com/tracing/manual_instrumentation/php)

安裝套件

`composer require datadog/dd-trace`

監控程式引用

請`use DDTrace\GlobalTracer;`

監聽段落

這部分寫在監聽的function內部，從想要監聽的開頭之上

`$scope = \DDTrace\GlobalTracer::get()->startActiveSpan('TrafficLimit.handle666666');`

監聽完成區段的結尾

`$scope->close();`

範例
```
public function handle($request, Closure $next)
    {
        //開始監聽
        $scope = \\DDTrace\\GlobalTracer::get()->startActiveSpan('TrafficLimit.handle666666');
        $route\_name         = $request->route()->getName();
        $redis\_key\_limit    = $route\_name.'\_limit';
        $dont\_save          = in\_array($route\_name, $this->dont\_save\_route\_name);

        if ($dont\_save == false) {
            $defaultGlobalTrafficLimit = config('mmrm.default\_global\_traffic\_limit', 100);
            $trafficLimit = config('trafficLimit.'.$route\_name, $defaultGlobalTrafficLimit);
            $luaForSettingMissingLimitKey = <<< LUA
if redis.call("exists",KEYS\[1\]) == 0 then
    redis.call("set",KEYS\[1\],ARGV\[1\])
end
LUA;
            Redis::connection('app\_traffic\_limit')->eval($luaForSettingMissingLimitKey, 1, $redis\_key\_limit, $trafficLimit);
            $luaForDecrementingLimitKey = <<< LUA
return tonumber(redis.call("get",KEYS\[1\])) > 0 and redis.call("decr",KEYS\[1\]) or -1
LUA;
            throw\_unless(
                Redis::connection('app\_traffic\_limit')->eval($luaForDecrementingLimitKey, 1, $redis\_key\_limit) >= 0,
                new RateLimitExceededException('API rate limit exceeded.')
            );
        }
        //監聽自訂塞入的內容
        $scope->getSpan()->setTag('註解標籤', '註解內容');
        //結束監聽
        $scope->close();
        return $next($request);
    }
```

加入註解

`$scope->getSpan()->setTag('註解標籤', '註解內容');`

web api apm塞程式部分紀錄
------------------

mmrm\_app\_api
--------------

middleware

1.  HeaderCheck
    
    1.  多request的內容tag
        
2.  Traffic:imit
