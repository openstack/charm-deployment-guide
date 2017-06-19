2. Edit the ``/etc/charms/charms.conf`` file and complete the following
   actions:

   * In the ``[database]`` section, configure database access:

     .. code-block:: ini

        [database]
        ...
        connection = mysql+pymysql://charms:CHARMS_DBPASS@controller/charms
