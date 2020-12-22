```c
#include <ngx_config.h>
#include <ngx_core.h>
#include <ngx_http.h>

static ngx_int_t ngx_http_test_post_config(ngx_conf_t *cf);
static void* ngx_http_test_create_loc_conf(ngx_conf_t *cf);

static ngx_int_t ngx_http_test_handler(ngx_http_request_t *r);

typedef struct {
    ngx_str_t                       test_str;
    int                             flag;
} ngx_http_test_conf_t;


static ngx_command_t ngx_http_test_commands[] = {
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

    return conf;
}

static ngx_int_t ngx_http_test_post_config(ngx_conf_t *cf) {
    ngx_http_handler_pt             *h;
    ngx_http_core_main_conf_t       *cmcf;


    cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);

    h = ngx_array_push(&cmcf->phases[NGX_HTTP_CONTENT_PHASE].handlers);
    if (h == NULL) {
        return NGX_ERROR;
    }

    *h = ngx_http_test_handler;
    return NGX_OK;
}

static void ngx_http_test_on_read_body_handler(ngx_http_request_t *r) {
    ngx_log_error(NGX_LOG_ERR, r->connection->log, 0, "============== ngx_http_test_on_read_body_handler ==================");

    ngx_str_t response = ngx_string("{\"result\": 0}");

    // 组装头部
    r->headers_out.status = NGX_HTTP_OK;
    r->headers_out.content_length_n = response.len;
    ngx_str_set(&r->headers_out.content_type, "application/json");

    // 发送头部
    ngx_http_send_header(r);

    // 组装 body
    ngx_buf_t *b = ngx_create_temp_buf(r->pool, response.len);
    ngx_memcpy(b->pos, response.data, response.len);
    b->last = b->pos + response.len;
    b->last_buf = 1;
    b->last_in_chain = 1;

    ngx_chain_t out;
    out.buf = b;
    out.next = NULL;

    // 发送响应
    ngx_http_output_filter(r, &out);
}

static ngx_int_t ngx_http_test_handler(ngx_http_request_t *r) {
    ngx_log_error(NGX_LOG_ERR, r->connection->log, 0, "============== ngx_http_test_handler ==================");

    ngx_http_read_client_request_body(r, ngx_http_test_on_read_body_handler);

    return NGX_DONE;
}
```
