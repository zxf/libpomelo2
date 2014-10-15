libpomelo2 API
===================

## 常量

- 版本信息 

``PC_MAJOR_VERSION`` major version
``PC_MINOR_VERSION`` minor version
``PC_REVISION`` revision

## 函数返回的错误码

libpomelolibpomelo 中使用的内存申请和释放函数，如果都设置为NULL的话，将会使用malloc/free
 platform， 平台，仅仅是一个标识作用, 用户可以传自己的平台参数

 pc_lib_init只需要调用一次，而且必须是在最初调用


void pc_lib_cleanup()
与pc_lib_init相对应，清理整个lib环境，放在所有的调用之后调用

void pc_lib_set_default_log_level(int level) 设置日志输出级别，设置为PC_LOG_DISABLE的话，将关闭日志输出


pc_client_t

这个结构不对外暴露任何成员，用户可以把其当作一个不透明的结构来使用

size_t pc_client_size()
  pc_client_t所占空间大小, 用户根据此值来申请合适的空间大小
int pc_client_init(pc_client_t* client, void* ex_data, const pc_client_config_t* config)
  初始化client，对于uv_tcp以及uv_tls的实现来说，在这个调用里会启动uv的事件循环。参数如下：
   client - 要初始化的client
   ex_data - client可以携带的额外信息，可以由用户自由使用
   config - client的配置信息，在初始化时，client内部会对*config做一个拷贝, config设置为NULL的话，会使用PC_CLIENT_CONFIG_DEFAULT
   返回值:
    PC_RC_INVALID_ARG, client值为NULL
    PC_RC_NO_TRANS, 没找到config中指定的transport
    PC_RC_ERROR, 其他一些错误
    PC_RC_OK, 初始化成功

    example:
       pc_client_t* client = (pc_client_t*)malloc(pc_client_size());
       int ret = pc_client_init(client, NULL, NULL);
       if (ret == PC_RC_OK) {
         // init ok
       }

void* pc_client_ex_data(const pc_client_t* client);
   client携带的额外信息的获取方法， 返回值为pc_client_init调用的时候的ex_data参数

const pc_client_config_t* pc_client_config(const pc_client_t* client);
   返回client的配置信息结构体，返回值为只读的


int pc_client_connect(pc_client_t* client, const char* host, int port, const char* handshake_opts);
   发起到服务端的连接,
   client - client 实例
   host - 服务器地址，可以是8.8.8.8, 也可以是域名www.mygame.com
   port - 连接端口
   handshake_opts - 握手时，用户可以上传的自定义信息，不使用的话，设置为NULL即可

   返回值:
      PC_RC_INVALID_ARG, 有不合法参数
      PC_RC_INVALID_STATE, client处于的状态不能发起connect
      PC_RC_ERROR, 其他错误
      PC_RC_OK, 发起连接成功

   当连接成功或者失败后，会通过事件的形式上报


int pc_client_disconnect(pc_client_t* client)
   主动断开到服务端的连接
   client - client实例
   返回值:
      PC_RC_INVALID_ARG, client为NULL 
      PC_RC_INVALID_STATE, client处于的状态不能发起connect
      PC_RC_ERROR, 其他错误
      PC_RC_OK, 发起断开成功

  当断连成功后，会通过事件的形式上报, 当断开成功后client处于INITED状态

int pc_client_cleanup(pc_client_t* client)
   清理client
   client - client实例
   返回值：
      PC_RC_INVALID_ARG, client为NULL 
      PC_RC_ERROR, 其他错误
      PC_RC_INVALID_THREAD, 如果transport使用的是内建的uv_tcp或者uv_tls的话，pc_client_cleanup将不能在uv loop线程中调用，否则将会返回PC_RC_INVALID_THREAD
      PC_RC_OK, 清理成功后，处于NOT_INITED状态, 可以被重新init



int pc_client_poll(pc_client_t* client)
    如果开启了poll模式的话，这个调用将会回调所有挂起的回调
    返回值：
      PC_RC_INVALID_ARG, client为NULL
      PC_RC_ERROR, client没开启poll模式
      PC_RC_OK, poll成功

int pc_client_state(pc_client_t* client);
   查询client的状态

int pc_client_conn_quality(pc_client_t* client);
   查询client的连接质量，如果底层的transport不支持的话，将会返回-1。对于内建的uv_tcp和uv_tls, 使用了心跳的rtt作为返回值

void* pc_client_trans_data(pc_client_t* client);

   底层transport的提供的内部数据，对于内建的uv_tcp和uv_tls，transport提供的数据使用方式如下:
   uv_tcp:
   uv_loop_t* loop = (uv_loop_t*)pc_client_trans_data(client);
uv_tls:
    void** internal_data = (void**)pc_client_trans_data(client);
    uv_loop_t* loop = internal_data[0];
    SSL* s = (SSL*) internal_data[1];





## event 

libpomelo2 统一了事件处理, 事件处理回调函数为
void (*pc_event_cb_t)(pc_client_t *client, int ev_type, void* ex_data,
                              const char* arg1, const char* arg2);
   client - 发生事件的client实例
   ev_type - 事件类型
   ex_data - 在安装event handler的时候，指定的额外数据，供用户自由使用

针对不同的事件， 参数arg1, arg2有不同的意义, libpomelo2 支持如下事件类型:

 * event handler callback and event types
 *
 * arg1 and arg2 are significant for the following events:
 *   PC_EV_USER_DEFINED_PUSH - arg1 as push route, arg2 as push msg
 *   PC_EV_CONNECT_ERROR - arg1 as short error description
 *   PC_EV_CONNECT_FAILED - arg1 as short reason description
 *   PC_EV_UNEXPECTED_DISCONNECT - arg1 as short reason description
 *   PC_EV_PROTO_ERROR - arg1 as short reason description
 *
 * For other events, arg1 and arg2 will be set to NULL.
 */
PC_EV_USER_DEFINED_PUSH 收到服务器端的push消息，arg1是push route， arg2 是push msg， 这两个参数在回调中都是只读
PC_EV_CONNECTED 连接成功， arg1, arg2均为NULL 
PC_EV_CONNECT_ERROR 连接错误，arg1为一个简短的错误描述, arg2 为NULL

PC_EV_CONNECT_FAILED 连接失败，当重连次数超限后，或者没有开启自动重连并且第一次连接发生错误时，arg1为简短的错误描述，arg2为NULL 
PC_EV_DISCONNECT 主动断开连接成功，arg1, arg2均为NULL 
PC_EV_KICKED_BY_SERVER 被服务器踢掉，arg1, arg2均为NULL
PC_EV_UNEXPECTED_DISCONNECT 非正常断连，arg1为错误描述，arg2为NULL 
PC_EV_PROTO_ERROR 协议解析出现了错误，arg1为错误描述，arg2为NULL 

PC_EV_INVALID_HANDLER_ID

int pc_client_add_ev_handler(pc_client_t* client, pc_event_cb_t cb,
        void* ex_data, void (*destructor)(void* ex_data));

安装event handler， 成功返回一个handler_id, 失败返回PC_EV_INVALID_HANDLER_ID
 参数: 
    client - client 实例
    cb - event handler的回调
    ex_data - 在回调中可以访问的额外数据，用户可以自由定义, 不使用的话，设置为NULL即可
    destructor - 针对用户自定义的ex_data, 如果在卸载event handler的时候，需要对ex_data做清理，那么destructor可传入此卸载函数，不使用的话，设置为NULL即可
int pc_client_rm_ev_handler(pc_client_t* client, int id)
  卸载一个event handler
  client - client实例
  id - 安装event handler时返回的一个合法的handler id

  返回值：
    总是返回PC_RC_OK, 如果是一个错误的handler id将什么也不会做



