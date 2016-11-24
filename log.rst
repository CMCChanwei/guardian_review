log相关
=======

1 邮件发送失败的时候log未记录
-----------------------------

::

    if 'success' == self.send_mail(mail_content=mail_content):
        LOG.info('send mail success at %s for over threshold' % datetime.datetime.now())
        self.over_check_time = datetime.datetime.now()

guardian.py文件的98行~99行之间，除了邮件发送成功记录log信息，邮件发送失败（返回!=success）的时候应该有必要记录error或warning的log信息



2 log的命名方式会导致log被覆盖问题
----------------------------------

::

    @staticmethod
    def match_pair(slave_node, spare_node):
        script = CONF.script_location
        cmd = u"python %s -v all %s %s > /var/log/guardian/%s_%s_match.log 2>&1" % \
              (script, slave_node, spare_node, slave_node, spare_node)
        LOG.info('execute [%s]' % cmd)
        result = os.system(cmd)
          return True if result == 0 else False

guardian.py文件的315行，以后若slave_node, spare_node相同log会被覆盖



3 状态不全
----------

::

    def _evacuate_predict(self, server):
        if server.status not in ALLOWED_EVACUATE_VM_STATE:
            LOG.info('vm info:[id:%s,name:%s,status:%s],status is not active or shutoff,ignore...' %
                   (server.id, server.name, server.status))
          return False

guardian.py文件的323行，除了active和shutoff状态的vm，还有error状态



4 值被写死了，实际根据配置文件来的
----------------------------------

::

    if 'off' != power_status:
        LOG.info('power status of %s is not off,power off it...' % node)
        ipmi_util.do_power_off()
        for i in range(1, CONF.ipmi_check_max_count+1):
            power_status = ipmi_util.get_power_status()
            if 'off' != power_status:
                LOG.info('power status of %s is not off,wait for %s seconds...' % (node, CONF.ipmi_check_interval))
                time.sleep(CONF.ipmi_check_interval)
            else:
                break
        power_status = ipmi_util.get_power_status()
        if 'off' != power_status:
            LOG.info('after 30s check,can not confirm power status of %s is off,will ignore evacuate...' % node)
            ret = 0 
    return ret

guardian.py文件的374行“30s”

::

    #ipmi查询电源是否已经关机的尝试次数
    ipmi_check_max_count = 3
    #ipmi查询是否已经关机的间隔,单位秒
    ipmi_check_interval = 10

若配置文件变了，不一定就是30s
