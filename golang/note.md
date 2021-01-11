## 细节笔记

### goroutine的使用注意事项:

子goroutine的panic会引起整个进程crash, 所以需要在goroutine函数写上defer recover, 注意recover只在defer语句生效

