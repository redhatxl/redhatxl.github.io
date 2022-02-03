# gin-validator参数校验

# 一 背景

在web开发中一个不可避免的环节就是对请求参数进行校验，通常我们会在代码中定义与请求参数相对应的模型（结构体），借助模型绑定快捷地解析请求中的参数，例如 gin 框架中的`Bind`和`ShouldBind`系列方法。本文就以 gin 框架的请求参数校验为例，介绍一些`validator`库的实用技巧。

gin框架使用[github.com/go-playground/validator](https://github.com/go-playground/validator)进行参数校验，目前已经支持`github.com/go-playground/validator/v10`了，我们需要在定义结构体时使用 `binding` tag标识相关校验规则，可以查看[validator文档](https://godoc.org/github.com/go-playground/validator#hdr-Baked_In_Validators_and_Tags)查看支持的所有 tag。

# 二 示例

```go
package main

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

type SignUpParam struct {
	Age        uint8  `json:"age" binding:"gte=1,lte=130"`
	Name       string `json:"name" binding:"required"`
	Email      string `json:"email" binding:"required,email"`
	Password   string `json:"password" binding:"required"`
	RePassword string `json:"re_password" binding:"required,eqfield=Password"`
}

func main() {
	r := gin.Default()

	r.POST("/signup", func(c *gin.Context) {
		var u SignUpParam
		if err := c.ShouldBind(&u); err != nil {
			c.JSON(http.StatusOK, gin.H{
				"msg": err.Error(),
			})
			return
		}
		// 保存入库等业务逻辑代码...

		c.JSON(http.StatusOK, "success")
	})

	_ = r.Run(":8999")
}
```

我们使用curl发送一个POST请求测试下：

```bash
curl -H "Content-type: application/json" -X POST -d '{"name":"q1mi","age":18,"email":"123.com"}' http://127.0.0.1:8999/signup
```

输出结果：

```bash
{"msg":"Key: 'SignUpParam.Email' Error:Field validation for 'Email' failed on the 'email' tag\nKey: 'SignUpParam.Password' Error:Field validation for 'Password' failed on the 'required' tag\nKey: 'SignUpParam.RePassword' Error:Field validation for 'RePassword' failed on the 'required' tag"}
```

从最终的输出结果可以看到 `validator` 的检验生效了，但是错误提示的字段不是特别友好，我们可能需要将它翻译成中文。



# 三 翻译验证信息

`validator`库本身是支持国际化的，借助相应的语言包可以实现校验错误提示信息的自动翻译。下面的示例代码演示了如何将错误提示信息翻译成中文，翻译成其他语言的方法类似。

```go
package main

import (
	"fmt"
	"net/http"

	"github.com/gin-gonic/gin"
	"github.com/gin-gonic/gin/binding"
	"github.com/go-playground/locales/en"
	"github.com/go-playground/locales/zh"
	ut "github.com/go-playground/universal-translator"
	"github.com/go-playground/validator/v10"
	enTranslations "github.com/go-playground/validator/v10/translations/en"
	zhTranslations "github.com/go-playground/validator/v10/translations/zh"
)

// 定义一个全局翻译器T
var trans ut.Translator

// InitTrans 初始化翻译器
func InitTrans(locale string) (err error) {
	// 修改gin框架中的Validator引擎属性，实现自定制
	if v, ok := binding.Validator.Engine().(*validator.Validate); ok {

		zhT := zh.New() // 中文翻译器
		enT := en.New() // 英文翻译器

		// 第一个参数是备用（fallback）的语言环境
		// 后面的参数是应该支持的语言环境（支持多个）
		// uni := ut.New(zhT, zhT) 也是可以的
		uni := ut.New(enT, zhT, enT)

		// locale 通常取决于 http 请求头的 'Accept-Language'
		var ok bool
		// 也可以使用 uni.FindTranslator(...) 传入多个locale进行查找
		trans, ok = uni.GetTranslator(locale)
		if !ok {
			return fmt.Errorf("uni.GetTranslator(%s) failed", locale)
		}

		// 注册翻译器
		switch locale {
		case "en":
			err = enTranslations.RegisterDefaultTranslations(v, trans)
		case "zh":
			err = zhTranslations.RegisterDefaultTranslations(v, trans)
		default:
			err = enTranslations.RegisterDefaultTranslations(v, trans)
		}
		return
	}
	return
}

type SignUpParam struct {
	Age        uint8  `json:"age" binding:"gte=1,lte=130"`
	Name       string `json:"name" binding:"required"`
	Email      string `json:"email" binding:"required,email"`
	Password   string `json:"password" binding:"required"`
	RePassword string `json:"re_password" binding:"required,eqfield=Password"`
}

func main() {
	if err := InitTrans("zh"); err != nil {
		fmt.Printf("init trans failed, err:%v\n", err)
		return
	}

	r := gin.Default()

	r.POST("/signup", func(c *gin.Context) {
		var u SignUpParam
		if err := c.ShouldBind(&u); err != nil {
			// 获取validator.ValidationErrors类型的errors
			errs, ok := err.(validator.ValidationErrors)
			if !ok {
				// 非validator.ValidationErrors类型错误直接返回
				c.JSON(http.StatusOK, gin.H{
					"msg": err.Error(),
				})
				return
			}
			// validator.ValidationErrors类型错误则进行翻译
			c.JSON(http.StatusOK, gin.H{
				"msg":errs.Translate(trans),
			})
			return
		}
		// 保存入库等具体业务逻辑代码...

		c.JSON(http.StatusOK, "success")
	})

	_ = r.Run(":8999")
}
```

同样的请求再来一次：

```bash
curl -H "Content-type: application/json" -X POST -d '{"name":"q1mi","age":18,"email":"123.com"}' http://127.0.0.1:8999/signup
```

这一次的输出结果如下：

```bash
{"msg":{"SignUpParam.Email":"Email必须是一个有效的邮箱","SignUpParam.Password":"Password为必填字段","SignUpParam.RePassword":"RePassword为必填字段"}}
```