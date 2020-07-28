# Créer un nom de domaine

![](https://user-images.githubusercontent.com/3251022/88677943-e81e8f00-d0bb-11ea-80e7-ef5b6232aa1e.png)

*   Créer un certificat

![](https://user-images.githubusercontent.com/3251022/88678251-40559100-d0bc-11ea-847f-f16f628d310d.png)

# Créer une EC2 contenant un serveur qui rediriger vers le port 80

Par exemple un serveur SonarQube ?

# Créer un target group par EC2

![](https://user-images.githubusercontent.com/3251022/88677402-3b441200-d0bb-11ea-8d00-40a650f9eead.png)

![](https://user-images.githubusercontent.com/3251022/88677475-557df000-d0bb-11ea-8334-66bbc50aa504.png)

# Créer un Load Balancer

![](https://user-images.githubusercontent.com/3251022/88677677-96760480-d0bb-11ea-9b85-5660db086ce0.png)

# Ajouter des règles sur le port 443 du load balancer pour rediriger les sous domaines par exemple vers les EC2 (View/edit rules)

![](https://user-images.githubusercontent.com/3251022/88679248-3ed89880-d0bd-11ea-8ab8-2a50b372597c.png)

# Configurer route 53 pour pointer vers le Load Balancer Créé

![](https://user-images.githubusercontent.com/3251022/88678589-a3dfbe80-d0bc-11ea-8cb8-8a677ebab5be.png)
