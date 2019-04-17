---
layout: post
title:  "怎样修改 rails 日志格式"
date:   2019-04-17 10:40:52 +0800
categories: rails
---
# 发现问题
- ruby 2.5.1
- rails 5.2.2.1
- graphql 1.9.3

在 rails + graphql 的使用过程中，当使用以下查询:

```graphql
{
  packages{
    name
  }
}
```

rails 应用会打印出以下日志:

```
Parameters: {"query"=>"{\n  packages{\n    name\n  }\n}", "variables"=>{"id"=>1}, "graphql"=>{"query"=>"{\n  packages{\n    name\n  }\n}", "variables"=>{"id"=>1}}}
```

体验不友好，希望能找到改变这个日志格式的方法.

#  寻找原因

rails 日志服务使用的是 ActiveSupport::Notifications + ActiveSupport::LogSubscriber. LogSubscriber 的文档中有如下使用例子:
```ruby
module ActiveRecord
  class LogSubscriber < ActiveSupport::LogSubscriber
    def sql(event)
      # 打印日志
      info "#{event.payload[:name]} #{event.payload[:sql]}"
    end
  end
end

# 订阅
ActiveRecord::LogSubscriber.attach_to :active_record

# 发布
ActiveSupport::Notifications.instrument("sql.active_record", sql: sql)
```

查看 attach_to 的源码
```ruby
def attach_to(namespace, subscriber = new, notifier = ActiveSupport::Notifications)
  @namespace  = namespace
  @subscriber = subscriber
  @notifier   = notifier

  subscribers << subscriber

  # Add event subscribers for all existing methods on the class.
  subscriber.public_methods(false).each do |event|
    add_event_subscriber(event)
  end
end
```
因此可以通过 `ActiveRecord::LogSubscriber.subscribers`来得到所有的subscribers
```
(byebug) ActiveRecord::LogSubscriber.subscribers.map &:class
[ActiveRecord::LogSubscriber, ActionController::LogSubscriber, ActionView::LogSubscriber]
(byebug) ActiveRecord::LogSubscriber.subscribers.map &:patterns
[["sql.active_record"], ["process_action.action_controller", "send_file.action_controller", "start_processing.action_controller", "logger.action_controller", "halted_callback.action_controller", "redirect_to.action_controller", "send_data.action_controller", "unpermitted_parameters.action_controller", "write_fragment.action_controller", "read_fragment.action_controller", "exist_fragment?.action_controller", "expire_fragment.action_controller", "expire_page.action_controller", "write_page.action_controller"], ["render_collection.action_view", "render_partial.action_view", "render_template.action_view", "logger.action_view"]]
```
排除 `ActiveRecord::LogSubscriber, ActionView::LogSubscriber` 可以在 `A ctionController::LogSubscriber `找到我们想要的源码
```
def start_processing(event)
  return unless logger.info?

  payload = event.payload
  params  = payload[:params].except(*INTERNAL_PARAMS)
  format  = payload[:format]
  format  = format.to_s.upcase if format.is_a?(Symbol)

  info "Processing by #{payload[:controller]}##{payload[:action]} as #{format}"
  info "  Parameters: #{params.inspect}" unless params.empty?
end
```
可以在这下手改变 graphql 的日志格式。

# 解决办法
在 initializer 中编写以下代码。
```
require "action_controller/log_subscriber"

module GraphqlLogSubscriber
  def start_processing(event=nil)
    return unless logger.info?

    payload = event.payload
    params  = payload[:params].except(*ActionController::LogSubscriber::INTERNAL_PARAMS)
    format  = payload[:format]
    format  = format.to_s.upcase if format.is_a?(Symbol)

    info "Processing by #{payload[:controller]}##{payload[:action]} as #{format}"
    return if params.empty?
    if payload[:controller] == 'GraphqlController' and payload[:action] == 'execute'
      info color("Graphql: ", ActiveSupport::LogSubscriber::MAGENTA, false)
      info color("#{params[:query]}\n#{params[:variables]}", ActiveSupport::LogSubscriber::MAGENTA, true)
    else
      info "  Parameters: #{params.inspect}"
    end
  end
end

ActionController::LogSubscriber.prepend GraphqlLogSubscriber
```

现在的日志:
```ruby
Started POST "/graphql" for 127.0.0.1 at 2019-04-01 00:27:50 +0800
Handling connection for 5432
   (5.7ms)  SELECT "schema_migrations"."version" FROM "schema_migrations" ORDER BY "schema_migrations"."version" ASC
  ↳ /Users/ztluo/.rbenv/versions/2.5.1/lib/ruby/gems/2.5.0/gems/activerecord-5.2.2.1/lib/active_record/log_subscriber.rb:98
Processing by GraphqlController#execute as HTML
Graphql:
{
  packages{
    name
  }
}
{"id"=>1}
  Package Load (9.8ms)  SELECT "packages".* FROM "packages"
  ↳ app/controllers/graphql_controller.rb:10
Completed 200 OK in 112ms (Views: 0.4ms | ActiveRecord: 71.3ms)
```

# 拓展

1. ActiveSupport::Notifications 何时发布 start_processing.action_controller 事件的?

    [ActionController::Instrumentation.process_action](https://github.com/rails/rails/blob/b75192845a6aa89b35c857e9f3de443ae1a0fbd5/actionpack/lib/action_controller/metal/instrumentation.rb)

2. ActionController::LogSubscriber.process_action 怎么被调用的?

    [ActiveSupport::Subscriber#finish](https://github.com/rails/rails/blob/159e58fc60d612c3d5bd524c9cee0e0922132478/activesupport/lib/active_support/subscriber.rb)
