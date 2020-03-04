

个人理解：网关是api的入口，nginx也可以做一个简单的网关，但是spring cloud gateway会更加容易扩展，比如可以直接在网关做一个权限校验，而且一个服务往往有多个机器，网关可以进行代理。

gateway使用的springwebflux，实际上就是netty。

zuul1使用的是io阻塞模型。

zuul2使用的netty，io多路复用。