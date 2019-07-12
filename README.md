# 多语言工具
> 根据输入的 `name`,`value`,`lineNum`在各个`strings.xml`里指定的行号增加字段,支持`Google翻译`

## 前置
- `Groovy`环境
- `Google翻译`Api的key
    > 收费!

## 预览
![预览](https://github.com/CSnowStack/MultiLanguageStringTool/blob/master/preview/preview.gif)

## 使用
- 把`addString.groovy`所在`文件夹`复制到项目的对应位置
- 编辑`addString.groovy`最后的方法调用
- cd app/groovy
- groovy addString.groovy



## 代码 [Github](https://github.com/CSnowStack/MultiLanguageStringTool)


```java
import groovy.json.JsonOutput
import groovyx.net.http.HttpResponseException
import groovyx.net.http.RESTClient
import groovy.transform.Field

/**
 * @Grab http://ifeve.com/groovy-grape/
 * 网络访问的库
 * https://github.com/jgritman/httpbuilder
 */
@Grab('org.codehaus.groovy.modules.http-builder:http-builder:0.7.1')
import static groovyx.net.http.ContentType.URLENC



@Field fileDirs = new File("../src/main/res")
@Field googleTranslationUrl = 'https://translation.googleapis.com/language/'
--请填入google翻译api的key
@Field googleTranslationKey = 'google翻译api的key'
/**
 * 使用google翻译 的源语言
 */
@Field sourceLanguage = 'en'
/**
 * 默认语言
 */
@Field defaultLanguage = 'en'

static def getInput(def name, def value) {
    def input = "    <string name=\"$name\">$value</string>"
    return input
}

static def appendWihLineNum(def list, def lineNum, def value) {
    //超过最大行数则添加到最后
     list.add(list.size > lineNum ? lineNum : list.size - 1, value)
}

/**
 * 添加字符串文件
 * @param name name
 * @param value 默认值 或 array
 * @param lineNumber 行号,超过最大,会add进倒数第二行
 * @param useGoogleTranslation 要求 value 必须是英语
 */
void appendStringArrays(def name, def value, def lineNumber, def useGoogleTranslation) {
    def fileIndex = 0
    def input = getInput(name, value)

    //找匹配 values.* 的文件或文件夹
    fileDirs.eachFileMatch(~/values.*/) {
        directory ->
            //找strings.xml
            directory.eachFileMatch(/strings.xml/) {
                // 文件 转 List<String>
                def lines = it.readLines()
                def directoryNameSplitLanguage = directory.name.split("-")
                //取当前文件夹对应的语言
                def language = directoryNameSplitLanguage.length == 1 ? defaultLanguage : directory.name.split("-")[1]

                //需要google翻译,同语言不翻译
                if (useGoogleTranslation && language != sourceLanguage) {

                    //填充的内容, will be url-encoded
                    def body = [q: value, target: language, format: 'text', source: sourceLanguage, key: googleTranslationKey]
                    def http = new RESTClient(googleTranslationUrl)
                    http.ignoreSSLIssues()
                    try {
                        def resp = http.post(path: 'translate/v2', body: body, requestContentType: URLENC,
                                headers: ['Content-Type': 'application/x-www-form-urlencoded'])

                        //填充翻译之后的文本
                        def result = resp.getData().data.translations[0].translatedText
                        appendWihLineNum(lines, lineNumber, getInput(name, result))
                        it.text = lines.join('\n')
                        return
                    } catch (HttpResponseException e) {
                        r = e.response
                        println("Success: $r.success")
                        println("Status:  $r.status")
                        println("Reason:  $r.statusLine.reasonPhrase")
                        println("Content: \n${JsonOutput.prettyPrint(JsonOutput.toJson(r.data))}")
                    }

                } else if (value instanceof List) {
                    input = getInput(name, value.get(fileIndex))
                }

                appendWihLineNum(lines, lineNumber, input)
                //assert [1, 2, 3].join('-') == '1-2-3'
                //修改文件的内容
                it.text = lines.join('\n')
                fileIndex++

            }
    }
}



```