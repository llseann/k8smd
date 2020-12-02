#### 方法一：
使用eval关键字,结合while read line进行替换
```
export FRONTEND_API_SERVER_HOST=${FRONTEND_API_SERVER_HOST:-"127.0.0.1"}
export FRONTEND_API_SERVER_PORT=${FRONTEND_API_SERVER_PORT:-"12345"}

echo "generate app config"
ls ${DOLPHINSCHEDULER_HOME}/conf/ | grep ".tpl" | while read line; do
eval "cat << EOF
$(cat ${DOLPHINSCHEDULER_HOME}/conf/${line})
EOF
" > ${DOLPHINSCHEDULER_HOME}/conf/${line%.*}
done
```
语法解析：
1. 统一给配置文件添加`.tpl`后缀，方便进行替换操作
2. 搜索后缀为`.tpl`的配置文件名,读取到`${line}`里面,结合循环进行依次重写
3. 使用`eval`关键词，将配置文件内容进行二次解析，替换其中的变量占位符为环境变量中的具体值
4. 注意`$(cat ${DOLPHINSCHEDULER_HOME}/conf/${line})`最外层`$`不能省


#### 方法二：
使用sed进行替换,试用与少量的参数替换
```
sed -i "s/FRONTEND_API_SERVER_HOST/${FRONTEND_API_SERVER_HOST}/g" /etc/nginx/conf.d/dolphinscheduler.conf
sed -i "s/FRONTEND_API_SERVER_PORT/${FRONTEND_API_SERVER_PORT}/g" /etc/nginx/conf.d/dolphinscheduler.conf
```