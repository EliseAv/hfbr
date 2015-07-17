===================================
High Frequency Backup and Retention
===================================

Backup important small files under a high frequency schedule,
avoiding duplicates, and pruning less relevant states as they get older.

Licensed under the `Apache License 2.0`__.

__ LICENSE.rst

------------
Installation
------------

1.  Make sure you have python3_ installed.
2.  Copy ``example_settings.py`` into ``settings.py``.
3.  Edit the newly created `Settings File`_ according to your needs.
4.  Add ``hfbrw.py`` to your crontab. My schedule is ``*/20 * * * *``, which means every 20 minutes.

.. _python3: http://python.org/

-------------
Settings File
-------------

The settings file is expected to have three variables: ``TARGETS`` and ``PLANS``, described below,
and ``LOGGING``, as defined_ by the Logging configuration module.

.. _defined: https://docs.python.org/3/library/logging.config.html

TARGETS
^^^^^^^

This is the heart of your settings.
In it you have a list of targets, each representing a backup and/or a retention task::

    TARGETS = [
        {
            'target_path': '/home/django/honesty/db.sqlite3',
        },
        {
            'backup_dir': '/home/backup/kindness',
            'retention_plan': 'important',
        },
        {
            'target_path': '/home/django/loyalty/db.sqlite3',
            'backup_dir': '/home/backup/loyalty',
            'retention_plan': 'meh',
        },
        {
            'target_path': '/home/django/generosity/db.sqlite3',
            'backup_dir': '/home/backup/generosity',
            'retention_plan': 'important',
            'prune': False,
        },
        {
            'target_path': '/home/django/laughter/db.sqlite3',
            'backup_dir': '/home/backup/laughter',
            'retention_plan': 'important',
            'pin': ('20150717-1155.sq3.bz2',),
        },
        {
            'target_path': '/home/django/magic/db.sqlite3',
            'backup_dir': '/home/backup/magic',
            'retention_plan': ((None, 10), ('month', 3)),
            'pin': (),
            'prune': False,
        },
    ]

Available settings are:

*   ``target_path``: Full path to the file to be backed up. It is read without any kind of locking or waiting.
    If not given, only the retention policy is performed at ``backup_dir``.

*   ``backup_dir``: Full path to the directory where the backups are stored.
    If not given, it is backed up in place, alongside the same directory of ``target_path``.

*   ``retention_plan``: Name or description of the retention plan.
    See PLANS_ for details. If not given, no files are pruned.

*   ``pin``: An enumeration of files that are not to be pruned.
    Pinned files fulfill the retention slots they fall in.

*   ``prune``: Set it to ``False`` to run the retention plan in pretend mode. Results go in the logs.

PLANS
^^^^^

This is a dictionary that contains descriptions of backup plans mapped by name, to be referenced by the targets::

    PLANS = {
        'important': (
            ('year', None),  # permanent yearly snapshots
            ('month', 9),  # 9 monthly snapshots
            (timedelta(weeks=1), 6),  # 6 weekly snapshots
            (timedelta(days=1), 5),  # 5 daily snapshots
            (timedelta(hours=1), 18),  # 18 hourly snapshots
            (None, 10),  # 10 latest snapshots
        ),
        'meh': (
            (None, 5),  # 5 latest snapshots
            (timedelta(days=1), 8),  # 8 daily snapshots
        ),
    }

A retention plan is described as an enumerable of retention slots.
A retention slot is a 2-tuple composed of *granularity*, and *quantity*.

*   *Granularity* is how far apart we're interested the backups to be, so that they remain relevant as they get older.
    It can be a ``timedelta`` as defined by the datetime_ module.
    There are also three special values: ``'year'``, ``'month'``, or ``None``. ``None`` means look at all backups.

*   *Quantity* is how many backups to keep when looking within that granularity.
    The special value ``None`` means forever_, and it's a useful quantity to give to your largest granularity.

Every time the retention plan is applied to a backup directory, it will run each slot on all files.
For each desireable slot it will try to find a file to be retained, and pin the earliest one if none is found.

Because we don't always make backups, we will only count the slots when at least one file can fulfill a slot.
For example, let's say your file only changes on weekdays, and you have 7 day retention.
No files will be generated on weekends, and to fulfill 7 days we'll need to go over to the previous week until
at least 7 files can be found, each in a different day. This way, 7 non-consecutive days will be retained.

It's recommended to define retention slots from the biggest granularity to the smallest.
The reason is that if you define slots from the smallest to the biggest,
you will lose your earliest backups because later backups will fulfill the same granularity.

.. _datetime: https://docs.python.org/3/library/datetime.html#timedelta-objects
.. _forever: https://www.youtube.com/watch?v=ofvJU3AFOOo

-------
Roadmap
-------

*   Maybe detect changes based on mtime and size? Checksum seems a bit overkill...
*   Detect sqlite databases and use their backup function.
*   Detect directories and ``tar`` them.

    *   Use GNU differential tar (``-g``) based on retention granularities. Will this work on non-Linux?

*   Ability to upload backups somewhere else.
