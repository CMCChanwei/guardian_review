逻辑问题
========

1 cfg.py文件参数类型问题
------------------------

::

    cfg.ListOpt('receiver_list',
                default='support1@xxx.com,support2@xx.com'),

cfg.py文件的93行，cfg.ListOpt应该是list，如：
    default=['support1@xxx.com', 'support2@xx.com']



2 阙值检查的方法的逻辑
----------------------

    def _over_check(self):

::

    def _over_check(self):
        ret = True
        latest_samples_len = len(self.latest_samples_names)
        if latest_samples_len > 0:
            block_samples = [s for s in self.latest_samples if s.volume != 1]
            actual_ratio = 1.0 * len(block_samples) / latest_samples_len
            block_number = len(block_samples)
            if (CONF.valve.number > 0 and block_number > CONF.valve.number) or \
                    (CONF.valve.ratio > 0 and actual_ratio > CONF.valve.ratio):
                # 发送告警邮件
                LOG.warning('count of ok nodes over threshold')
                if CONF.email.enable_send_mail == 1 and (
                                self.over_check_time is None or self.over_check_time < self.get_before_time(
                                    CONF.email.send_interval_hours * 60 * 60)):
                    mail_content = u"检测节点异常超出阈值[阈值数为: %s, 节点异常数为: %s, 阈值比率为: %s," \
                                   u"实际比率为: %s ]，需人工确认处理" % \
                                   (CONF.valve.number, block_number, CONF.valve.ratio, actual_ratio)
                    if 'success' == self.send_mail(mail_content=mail_content):
                        LOG.info('send mail success at %s for over threshold' % datetime.datetime.now())
                        self.over_check_time = datetime.datetime.now()
                else:
                    LOG.info("ignore send mail for threshold:have send mail in recent %s hours"
                             % CONF.email.send_interval_hours)
                ret = False
        return ret

guardian.py文件的78行~102行，这个检查阙值的函数好像没有考虑下面两只情况

#. 节点报不通的数量以及不同的比例发生变化的情况（两次都在阙值以上，但这两个值发生了变化）
#. 超过阙值 -> 降到阙值以下 -> 再次超过阙值

以上这两种情况也是要经过max_report_error_count = 3（3小时）发一次邮件告警，感觉这种情况在执行_over_check的时候应该再次发邮件告警，而不是经过3小时



3 死循环问题
------------

::

     def _wait_for_service_to_down(self, node):
         ret = 0
         while True:
             try:
                 service = self.nova_client.services.list(host=node, binary='nova-compute')
                 if service:
                     if service[0].state != 'down':
                         LOG.info('service state of %s is %s, wait for it to down' % (node, service[0].state))
                         time.sleep(5)
                     else:
                         LOG.info('service state of %s is down now' % node)
                         ret = 1
                         break
             except Exception as e:
                 LOG.error('get service with node %s failed' % node)
                 LOG.error(e)
         return ret

guardian.py文件的408行~424行，若一直不能将该node down掉，则陷入死循环，感觉应该有个超时或什么处理机制
