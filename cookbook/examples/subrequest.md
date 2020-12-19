```c
#include <ngx_config.h>
#include <ngx_core.h>
#include <ngx_http.h>

static ngx_int_t ngx_http_test_post_config(ngx_conf_t *cf);
static void* ngx_http_test_create_loc_conf(ngx_conf_t *cf);

static ngx_int_t ngx_http_test_handler(ngx_http_request_t *r);
static char *ngx_http_mytest_callback(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);

typedef struct {
    ngx_str_t                       test_str;
    int                             flag;
    int                             count;
} ngx_http_test_conf_t;

int flag;


static ngx_command_t ngx_http_test_commands[] = {
    { ngx_string("mytest"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_CONF_TAKE12,
      ngx_http_mytest_callback,
      NGX_HTTP_LOC_CONF_OFFSET,
      0,
      NULL },
    ngx_null_command
};

static ngx_http_module_t ngx_http_test_module_ctx = {
    NULL,
    ngx_http_test_post_config,
    NULL,
    NULL,
    NULL,
    NULL,
    ngx_http_test_create_loc_conf,
    NULL
};

ngx_module_t ngx_http_test_module = {
    NGX_MODULE_V1,
    &ngx_http_test_module_ctx,
    ngx_http_test_commands,
    NGX_HTTP_MODULE,
    NULL,
    NULL,
    NULL,
    NULL,
    NULL,
    NULL,
    NULL,
    NGX_MODULE_V1_PADDING
};

static void * ngx_http_test_create_loc_conf(ngx_conf_t *cf)
{
    ngx_http_test_conf_t  *conf;

    conf = ngx_pcalloc(cf->pool, sizeof(ngx_http_test_conf_t));
    if (conf == NULL) {
        return NULL;
    }

    conf->flag = 0;
    conf->count = 0;
    flag = 0;

    return conf;
}

static ngx_int_t ngx_http_test_post_config(ngx_conf_t *cf) {
    ngx_http_handler_pt             *h;
    ngx_http_core_main_conf_t       *cmcf;


    cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);

    h = ngx_array_push(&cmcf->phases[NGX_HTTP_ACCESS_PHASE].handlers);
    // h = ngx_array_push(&cmcf->phases[NGX_HTTP_CONTENT_PHASE].handlers);
    if (h == NULL) {
        return NGX_ERROR;
    }

    *h = ngx_http_test_handler;
    return NGX_OK;
}

static char * ngx_http_mytest_callback(ngx_conf_t *cf, ngx_command_t *cmd, void *conf) {
    return NGX_CONF_OK;
}

static ngx_int_t
ngx_http_foo_subrequest_done(ngx_http_request_t *r, void *data, ngx_int_t rc)
{
    char  *msg = (char *) data;

    ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                  "============= done subrequest r:%p msg:%s rc:%i", r, msg, rc);

    ngx_http_test_conf_t       *cf;

    cf = ngx_http_get_module_loc_conf(r->main, ngx_http_test_module);
    cf->count += 1;
    flag += 1;
    return rc;
}

static ngx_int_t ngx_http_test_handler(ngx_http_request_t *r) {

    ngx_http_test_conf_t       *cf;

    cf = ngx_http_get_module_loc_conf(r, ngx_http_test_module);

    ngx_log_error(NGX_LOG_ERR, r->connection->log, 0, "============== ngx_http_test_handler ================== flag: %d, r: %p, cf: %p, count: %d", flag, r, cf, cf->count);

    cf->count += 1;

    if (flag == 2) {
        return NGX_OK;
    }

    ngx_str_t                   uri;
    ngx_http_request_t          *sr;
    ngx_http_post_subrequest_t  *ps;

    /*
    子请求处理完后回调
    */
    ps = ngx_palloc(r->pool, sizeof(ngx_http_post_subrequest_t));
    ps->handler = ngx_http_foo_subrequest_done;
    ps->data = "foo";

    /*
    子请求路径
    */
    ngx_str_set(&uri, "/foo");

    // return ngx_http_subrequest(r, &uri, NULL, &sr, ps, NGX_HTTP_SUBREQUEST_IN_MEMORY);
    ngx_http_subrequest(r, &uri, NULL, &sr, ps, NGX_HTTP_SUBREQUEST_IN_MEMORY);
    return NGX_AGAIN;
}
```
