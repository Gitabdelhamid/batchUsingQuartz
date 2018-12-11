# batchUsingQuartz
Exemple Quartz et Spring Scheduling

Quartz et Spring Scheduling
Quartz pour ceux qui ne le connaissent pas encore, est un ordonnanceur. Il permet de planifier des tâches pour des exécutions ponctuelles ou répétées. Les planifications possibles vont de la simple répétition infinie, à la répétition calendaire utilisant la syntaxe de cron (tous les jours à minuit, le 31 janvier 2009 à 12h00, …). Quartz est prévu pour toutes sortes d’applications allant des programmes standalone basiques aux gros systèmes JEE distribués.

De son côté, Spring scheduling nous donne le choix entre les Timers Java et Quartz. Avec ses interfaces, Spring nous permet de planifier, très facilement, des Jobs directement dans l’ApplicationContext. En revanche, la gestion des planifications dynamiques est moins évidente. Dans cet article, je vous présenterai l’API Quartz dans les grandes lignes, puis on mettra en place la planification Quartz avec Spring Scheduling.


Présentation de Quartz
Quartz est sorti de l’ombre en 2005 au moment de son intégration dans le projet OpenSymphony. C’est depuis devenu l’API de référence pour répondre à des besoins de planification et d’exécution de tâches répétitives. Fort de ses années d’existence et des nombreux retours utilisateurs, Quartz s’adresse à tout type d’application : aux petites applications dans une configuration minimale avec quelques Threads, comme aux gros systèmes avec des centaines de Threads répartis sur plusieurs machines.

Fonctionnalités
Planification fournissant un paramétrage très fin. On peut faire des tâches ponctuelles à une date précise, des tâches répétitives avec un nombre de répétitions paramétré et un délai entre chaque appel, ou bien utiliser une expression cron pour des planifications plus précises.
Séparation entre les tâches exécutables (« Jobs ») et les planifications (« Triggers »). On en tire une grande souplesse dans la gestion de l’ordonnancement.
Gestion d’erreur. Les Jobs peuvent remonter des JobExecutionException qui indiquent le comportement à adopter par le Scheduler. On peut relancer le Job immédiatement, supprimer le trigger qui l’a invoqué ou supprimer tous les Triggers. On peut aussi définir un comportement pour une planification ratée (ie. pas exécutée à la bonne date).
Jobs avec ou sans états. Quartz gère les données afférentes aux tâches dans un JobDataStore qui permet à chacune des tâches de récupérer des paramètres et/ou de modifier des données de son contexte. Pour chaque Job, on peut spécifier si les données doivent être maintenues entre chaque appel ou non. Notez bien qu’un Job StateFull ne peut s’exécuter en mode parallèle.
Persistance des Jobs. On peut configurer Quartz pour sauvegarder en base de données toutes les planifications, afin de pouvoir redémarrer l’application sans perte.
Mode client / serveur. Le Scheduler peut exposer ses services en RMI à destination d’autres Schedulers qui fonctionneront en proxy et reporteront leurs appels au serveur.
Support des transactions JTA. Quand la persistance est activée, on peut configurer l’ordonnanceur pour participer aux transactions.
Mise en cluster. En mode persistant, on peut configurer plusieurs Schedulers en cluster. Au moment où un Trigger doit se déclencher, un Scheduler pose un verrou dessus et le traite, ce qui répartit naturellement la charge. Pour la reprise sur erreur, quand une instance tombe, les autres vont récupérer les Jobs qui lui étaient affectés et feront tourner ceux marqués pour recouvrement (« request recovery »).

Comment ça marche ?
Le principe est de récupérer une instance de Scheduler configurée par une factory. On peut ensuite y ajouter les jobDetails qui décrivent les jobs qui seront exécutés. Le job est un objet qui implémente l’interface org.quartz.Job et fournit une méthode execute qui sera invoquée pour effectuer le boulot. Une fois que le ou les jobs sont ajoutés au Scheduler, il faut les planifier pour qu’ils soient invoqués quand on le souhaite. On ajoute donc des Triggers qui référencent un job en particulier.

Un SimpleTrigger permet de fournir une date d’exécution, un nombre de répétitions et un nombre de secondes à attendre entre chaque lancement. Si cela ne suffit pas, on peut utiliser un CronTrigger qui implémente une planification par crontab, où l’on spécifie les secondes, les minutes, les heures, les mois, les jours du mois et les jours de semaine d’exécution. Lorsqu’un Trigger arrive à terme, le Scheduler prend un Thread dans le pool pour instancier et exécuter le Job.

Tout commence toujours par le Job :

public class MyJob implements Job {
		public void execute(JobExecutionContext ctx)
				throws JobExecutionException {
			System.out.println("execution du job: " + ctx.get("message"));
		}
	}
Ensuite vient la planification du Job :

//on initialise le scheduler et le démarre
Scheduler sched = StdSchedulerFactory.getDefaultScheduler();
sched.start();

//on construit le JobDetail pour MyJob
JobDetail jobD = new JobDetail("myJob", null, MyJob.class);

//Le premier Trigger doit se déclencher le premier de chaque mois à minuit
Trigger trigger = TriggerUtils.makeMonthlyTrigger(1, 0, 0);
trigger.getJobDataMap().put("message", "First Trigger");
sched.scheduleJob(jobD, trigger);

//Le deuxième Trigger doit se déclencher tous les jours des mois de février et mars à minuit
CronTrigger trig = new CronTrigger();
trig.setCronExpression("0 0 0 2-3 ?");
trig.setJobName(jobD.getName());
trig.getJobDataMap().put("message", "Second Trigger");
sched.scheduleJob(trig);
Spring à la rescousse ?
Quand on veut utiliser Quartz avec Spring, tout paraît très simple. Il suffit de monter un bean SchedulerFactoryBean et de lui injecter des Jobs ou directement les Triggers qui les référencent.
Le MethodInvokingJobDetailFactoryBean permet d’appeler directement une méthode d’un POJO en ciblant la classe ou le Bean (au sens Spring) et le nom de la méthode à invoquer pour le Job. On pourra préciser l’identité qu’aura le Job dans le Scheduler et s’il doit être avec ou sans état. Attention tout de même, les fonctions liées à la persistance ne sont pas supportées par ce type de Job.

De leur côté, les Cron et SimpleTriggerBean permettent de configurer les Triggers pour référencer des jobDetails et les injecter dans la factory. C’est, en clair, l’option tout XML où vous configurez votre planification entièrement par l’ApplicationContext.

Bien que cette méthode soit bien documentée dans Spring, voici un petit exemple pour illustrer mes propos.

On a donc un bean pour faire le job :

 public class MonJobPojo {
	private DumbManager monManager;

	public void executeJob(){
		System.out.println("execution du job: " + monManager.doSomething());
	}

	public void setMonManager(DumbManager monManager) {
		this.monManager = monManager;
	}
}
Et la configuration de l’ApplicationContext en XML:

<!-- bean contenant la méthode a lancer régulièrement -->
 <bean id="monJobPojo" class="fr.xebia.quartz.job.monJobPojo">
 <property name="monManager" ref="monManager" />
 </bean>
 
 <!-- encapsulation dans un JobDetail -->
 <bean id="monJobPojoJobDetail"
 class="org.springframework.scheduling.quartz.MethodInvokingJobDetailFactoryBean">
 <property name="targetObject" ref="monJobPojo" />
 <property name="targetMethod" value="executeJob" />
 </bean>
 
 <!--
 Trigger simple, démarrage après 10s et se lance toutes les minutes
 -->
 <bean id="monJobPojoSimpleTrigger" class="org.springframework.scheduling.quartz.SimpleTriggerBean">
 <property name="jobDetail" ref="monJobPojoJobDetail" />
 <property name="startDelay" value="1000" />
 <property name="repeatInterval" value="25000" />
 </bean>
 
 <bean id="monJobPojoCronTrigger" class="org.springframework.scheduling.quartz.CronTriggerBean">
 <property name="jobDetail" ref="monJobPojoJobDetail" />
 <property name="cronExpression" value="0 0 0 * 2-3 ?" />
 </bean>
 
 <!-- SchedulerFactoryBean pour gérer le tout -->
 <bean class="org.springframework.scheduling.quartz.SchedulerFactoryBean">
 <property name="triggers">
 <list>
 <ref bean="monJobPojoSimpleTrigger" />
 <ref bean="monJobPojoCronTrigger" />
 </list>
 </property>
 </bean>
Tout ceci peut s’avérer très pratique lorsque l’on connaît à l’avance nos Triggers et nos Jobs, par exemple pour une purge s’exécutant tous les jours à 2h du matin. Maintenant quand on veut pouvoir ajouter des Jobs et/ou des planifications dynamiques, cela paraît plus compliqué. D’autant que la documentation de Spring est assez pauvre sur le sujet.

Un peu de dynamisme
Pour pouvoir tirer entièrement parti de Quartz avec l’ami Spring, il suffit de monter une classe ServiceScheduler qui se chargera de planifier vos tâches dynamiques à la demande en lui injectant le SchedulerFactoryBean dans un champ de type Scheduler. Votre classe possède maintenant l’instance du Scheduler Quartz configurée par Spring. Libre à vous ensuite d’y ajouter tous les Jobs et Triggers nécessaires.

Avoir un accès direct au Scheduler est une bonne chose, mais qu’en est-il des Jobs ? La plupart du temps, on a une liste de Jobs déjà implémentés, qui peuvent être planifiés. On va donc pouvoir injecter les JobDetails construits par Spring dans notre ServiceScheduler ou directement dans le Scheduler.

Si par ailleurs vos Jobs ont des traitements paramétrés, ou que vous avez besoin de la persistance, vous devrez cette fois implémenter QuartzJobBean et sa méthode executeInternal. Elle se chargera de propager les attributs du contexte d’exécution vers les propriétés de votre classe. Ainsi, au moment de l’ajout d’un Trigger, on devra prendre soin de passer les bons paramètres au contexte.

On a donc un QuartzJobBean :

public class MonJobBean extends QuartzJobBean
{
	private DumbManager monManager;
	private String message;

	public void setMonManager(DumbManager monManager) {
		this.monManager = monManager;
	}

	public void setMessage(String message){
		this.message = message;
	}

	protected void executeInternal(JobExecutionContext arg0)
			throws JobExecutionException {
		System.out.println("execution du job: " + message);
		monManager.doSomething();
	}
}
Le SchedulerService :

private Scheduler scheduler;
private JobDetail myJob;

public void addMyJobExecution(String cronExp){
	CronTrigger trig = new CronTrigger();
	try {
		trig.setCronExpression("0 0 0 * 2-3 ?");
		trig.setJobName(myJob.getName());
		trig.getJobDataMap().put("message", "CronTrigger");
		scheduler.scheduleJob(trig);
	} catch (ParseException e) {
		e.printStackTrace();
	} catch (SchedulerException e) {
		e.printStackTrace();
	}
}

public void setScheduler(Scheduler scheduler) {
	this.scheduler = scheduler;
}

public void setMyJob(JobDetail myJob) {
	this.myJob = myJob;
}
Et le contexte XML:

<bean name="exampleJob" class="org.springframework.scheduling.quartz.JobDetailBean">
 <property name="jobClass" value="fr.xebia.quartz.job.MonJobBean"/>
 <property name="jobDataAsMap">
 <map>
 <entry key="monManager" ref="monManager"/>
 </map>
 </property>
</bean>
 
<bean id="quartzScheduler" class="org.springframework.scheduling.quartz.SchedulerFactoryBean"/>
 
<bean name="schedulerService" class="fr.xebia.quartz.SchedulerService">
 <property name="scheduler" ref="quartzScheduler"/>
</bean>
Pour ceux que la persistance intéresse, Spring permet d’injecter directement une DataSource dans son SchedulerFactoryBean. Le Scheduler utilisera alors votre source de données et le pool qui va avec, plutôt qu’un pool interne à Quartz. En prime, les opérations de persistances pourront prendre part aux transactions gérées par Spring.

Les nouvelles du front
La dernière version stable est la 1.6.4 sortie le 23/11/2008. Cette version n’embarque que des corrections de bugs. Pour vous donner une idée, la version 1.6, qui est la dernière réelle évolution de l’API avec l’exposition du Scheduler en JMX et quelques helpers pour lancer des messages JMS, date de juin 2006. Quartz semble donc s’être quelque peu reposé sur ses lauriers en oubliant d’évoluer et de tirer parti des apports de Java 5 par exemple.

James House, le leader du projet depuis le début, confirme que, par manque de temps, Quartz est en mode « maintenance » pour le moment. Des modifications importantes qui pourraient même rompre la compatibilité ascendante sont cependant prévues. L’idée est clairement de moderniser cette API vieillissante.

Questionné au sujet des Service Time qui pourraient intégrer JEE6, James répond qu’il ne croit pas trop en cette énième tentative de spécification mais qu’il l’implémentera avec plaisir si elle finit par sortir. Il ajoute tout de même avec un brin d’ironie: « Si elle suffit aux utilisateurs. »

James se donne les six prochains mois pour réaliser tout ça ; le temps jugera.

Conclusion
Quartz, c’est bien parce que ça fonctionne et qu’il n’y a pas de réelle alternative. L’API est simple et efficace bien qu’on ne comprenne pas toujours ses choix. Par exemple, l’implémentation des Jobs StateFull, dont les instances ne sont pas maintenues, peut surprendre. On aurait aussi aimé voir une ou plusieurs annotations pour simplifier la création des Jobs. On l’a déjà dit, mais l’API est vieillissante. On sait maintenant que le staff du projet en est conscient. Espérons donc que la nouveauté ne tarde pas trop à arriver malgré un planning serré.

De son côté, Spring apporte de réelles améliorations dans l’utilisation de l’API et fournit bien sa valeur ajoutée. Cependant, avec les restrictions imposées et la faiblesse de la documentation, on ne peut s’empêcher de trouver qu’ils n’ont pas été au bout du travail. Passée la difficulté de la planification dynamique, on peut dire que l’association de Spring avec Quartz est assez fructueuse.

Attention toutefois a ne pas confondre Quartz avec un système de traitement par lot comme Spring Batch. Quartz ne s’intéresse qu’à la planification fine et l’exécution ou l’appel de traitement en masse. Pour des traitements lourds, on pourra par exemple lancer les batchs réalisés avec Spring à l’aide d’un Job Quartz.
