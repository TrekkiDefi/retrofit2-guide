|BaseUrl                                 |和URL有关的注解中提供的值          |最后结果|
|----------------------------------------|-----------------------------------|--------|
|http://localhost:4567/path/to/other/    |/post                           |http://localhost:4567/post|
|http://localhost:4567/path/to/other/    |post                         	|http://localhost:4567/path/to/other/post|
|http://localhost:4567/path/to/other/    |https://github.com/ittalks       |https://github.com/ittalks|

从上面不能难看出以下规则：

如果你在注解中提供的url是完整的url，则url将作为请求的url。
如果你在注解中提供的url是不完整的url，且不以 / 开头，则请求的url为baseUrl+注解中提供的值
如果你在注解中提供的url是不完整的url，且以 / 开头，则请求的url为baseUrl的域名部分+注解中提供的值