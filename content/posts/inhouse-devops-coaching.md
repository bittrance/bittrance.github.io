---
title: In-house cloud developer coaching for IT staff
date: 2024-05-26T12:32:00+02:00
draft: false
summary: >
authors:
  - Anders Qvist
tags:
  - devops
  - coaching
keywords:
  - devops
  - coaching
---
My current role is as a developer at a cloud solution provider. The company mostly works with cloud infrastructure and cloud services consultancy. Our main expertise is on virtual machines, networking and related security, but like most IT staff, my colleagues also know a bit of programming and serverless. Many share an ambition to automate and standardize the services they provide, but without experience in development methodology, they struggle to get beyond the point where they are copying scripts around.

We can see that we need to get better att software development as a company. We are currently performing a lot of lift operations, but if we are to remain relevant to our customers when they switch to shift, we need developers. Also, public clouds provide a very good toolbox for integration and automation, which allows us to demonstrate our value as a partner.

I therefore took it upon myself to start a coaching program for those willing to learn more about developing and operating services. Today's public clouds offer a rich palette of serverless technologies which make it realistic for a single developer or small team to develop and deploy simple services such as integrations. The goal of the program was to enable and empower my students to take their ideas to production.

I had a limited amount of time for face-to-face coaching: I could allocate about an hour per week per student. There are a lot of teaching resources out there already, so my initial ambition was to put together a curated set of resources and programmes that my students could follow and I could focus on assisting them. These resources fall broadly in two categories:

- Tool or technology-specific courses, tutorials and walk-throughs. These focus on teaching a technology such as Terraform or a narrow section of development such as unit testing. The assumption is that you are already professional and want to improve your skill in some particular area.
- Foundational developer training programs. These focus on giving a broad understanding of programming and developer work. The assumption is that you will work in a context where experienced colleagues will complement your skill set, particularly in getting services to production and operating them.

However, my students were neither professional developers nor working alongside senior developers. Thus, while such resources are helpful, they give no guidance as to what skills beyond coding the students need in order to succeed. Furthermore, they often gloss over important aspects which will teach students bad habits.

I needed a map (or a bingo chart, as one colleague called it) that allowed the students to follow their own path, but still be confident that they cover all the pieces that were needed to take their ideas to production. In other words: **what does it take to be a one-man cloud development team?**

The best attempt to answer this question that I have found is The DevOps roadmap, but it is a bit too focused on the devops role, rather than devops culture. It also presents the journey as a in terms of personal improvement, mastering one technology at a time.

## A model for devops learning

This insight led me to the conclusion that my students needed a breadth-first-first approach. This approach also facilitated management approval - the focus on capability-building ensured that students would be more productive in their current roles. I came up with this model:

![Devops learning map](/inhouse-devops-coaching/map.png)

The idea is that in order to progress to the right in one area in the picture, you need to be familiar not only with the skills in the same color band, but also those areas below. You could drill down into any of these areas independently (the DevOps Roadmap approach), but it would not make you more proficient in taking your ideas to production.

**Thus, this model rests on the premise that getting projects running in production and creating value takes precedence over building any particular technical skill.** We do not set topical learning goals. Instead the student is encouraged to take multiple trips across the map. It is the responsibility of the coach to ensure that the student's understanding in each area is deepened with each trip. The first few trips can be done with "toy" projects, but the earlier projects can facilitate the student's daily work, the better.

The model itself thus serves two main purposes:

- It helps students and coaches to reason about the feasibility of the student taking on a particular project, and
it serves as a reminder for the student not to spend too much time on a particular area, but to focus on the goal of getting ideas to production.
- By extension, the map serves to orient the student about most of the elements that go into modern software development, thus increasing their ability to address a wide variety of challenges that will arise in complex projects.

## Getting started

Before getting started, the student should pick a technology for each of these areas:

- a programming language
- a version control provider
- an execution environment
- an IaC solution

The picks should be informed by the area the student's general area of interest. For example, a student with an ambition to work with machine learning should probably pick Python as their programming language.

Students will probably have to learn several different technologies in each category during their training, but picking defaults early on will increase the chance that the student masters at least one. In many cases, most technologies in a category will serve; better the student uses a familiar technology.

See below for some recommendations on what technologies to pick. Remember that this article is colored by our context as a cloud solution provider.

## Coaching

I have mainly used three different coaching methods:

- **1:1 meetings**: traditional mentoring sessions allows us to plan projects, follow progress and refer back to the map to ensure that all areas are covered.
- **Pull request reviewing**: as a coach I ask students to make changes to their software and infrastructure using pull requests that I can then review. This gives the student reason to follow a workflow. It also gives the coach an opportunity to correct minor things that you don't want to waste time on in face-to-face sessions.
- **Pair programming**: occasionally, students get stuck or are unsure about how to solve particular problems. I encourage them to seek help and usually let them drive and explain what they are doing. These are good opportunities to train reasoning about software.

My agreement with management is for a fixed budget of 1 hour per student per week.

## Programming

There are many possible programming languages that students can work with, and selection depends on context, but Python, Go and Node.js are good candidates. They all have rich ecosystems and are easy to get started with.

One student picked Powershell, which worked well in the early stages, but its ecosystem turned out to bit too anemic to be able solve a wider range of problems.

- **Coding**: The basic skill of writing software: learning about condition, sequence and iteration. Most tech people have some proficiency in this area, but understanding is often patchy. Most of my students need to improve their computational thinking, particularly in working with patterns and abstractions.
- **Ecosystem**: All modern programming languages come with large open source package repositories. In Node.js or Rust, a project can easily have 100 direct and indirect dependencies, each focusing on doing one thing really well. This has made all modern software development into an exercise in integration. The student will need to learn how to discover, evaluate, use and eventually publish packages.
- **Structure and refactoring**: Once we know the basics of coding and leveraging the ecosystem, the main challenge is to shape code into a clear and concise narrative about how it achieves its goal. As the student evolves their project to achieve new goals, they need to learn how to structure it so that it remains maintainable. It needs to be reshaped for the narrative to remain true. This is known as refactoring. Further along, the student needs to know how to extract supporting libraries and find new abstractions.
- **Test-driven development**: Simple software can be tested by simply building and executing it. But as the project evolves, it will have to contend with ever more special cases. This means the student need a method to verify that the code continues to perform as expected even as we evolve it. While strict adherence to test-driven development is usually overkill, it remains a useful teaching technique. More generally, the student needs to understand how to automate tests and test setup.
- **OS or Browser fundamentals**: Modern frameworks abstracts away most of the interaction with the underlying operating system (or browser in the case of web frontends), but the student will still need some understanding of how to interact with the environment, e.g. how file systems work and how memory is managed. By extension, this builds the student's intuition for what can be expected in terms of services, parallelism and performance.
- **Specialization**: At some point the student will need to put more emphasis on some subdomain of programming. This may be something like frontend, stream processing, data science, embedded or similar. Such disciplines come with their own mental models, lingo and tools that the student needs to master. Beyond the specialized skills of the chosen subdomain, this builds awareness that problems have been encountered and solved before.

## Workflows

Unless you have special circumstances, the student should be using Git for version control, since it dominates the market. One part of this is picking a version control provider. I recommend GitHub, which has a strong feature set. Also, the ability to host personal projects for free is very useful for students.

Students should also pick a workflow. Because pull requests are a good teaching tool and foster discipline, I recommend trunk-based development with short-lived feature branches.

- **Version control**: The student needs to learn how to work with version control to make, manage and review changes. Beyond that, version control underpins releasing and debugging.
- **Developing alone**: The student needs to learn the basics of breaking work into manageable tasks that can be coded and verified one at a time. This requires planning and discipline, necessary for attacking large and complex projects.
- **Continuous integration**: Source control platforms allow us to continuously verify that the code follows our policies, builds and that the tests pass. The student will need to learn how to configure pipelines and how those interact with other source control platform features.
- **Developing together**: Once the student has learned a basic development workflow, it is time to learn how to adapt this workflow to cooperate with others. Ideally, this includes both working with other students or professional developers, and contributing to open source projects.
- **Testing strategies**: Knowing what to test and when is a very important skill for students to build. This extends familiarity with test-driven development with strategies for ensuring that the project is correct when it is deployed at production. This includes deciding what to include and what to mock, test data management and leveraging multiple environments for overlapping verification.

## Software in production

Students should focus on one execution environment at least for the first few months. There are many to pick from, but Azure Container Apps and AWS Fargate stand out for their versatility and cost efficiency. Ideally, the employer helps with covering students' public cloud costs through the cloud provider's developer program. Be wary of function-as-a-service frameworks which tend to foster bad practices by abstract away too many aspects of development and deployment.

Students aiming for data engineering may want to look at data platforms such as Azure Fabric, but be advised that these platforms present challenges in other areas, such as version control and testing.

- **Execution environment**: The student will need to understand the chosen execution environment, its strengths and weaknesses and how to operate services there. In the first few iterations, the student will probably deploy manually to the environment. As projects grow more complex, the student will also need to learn how to translate more complex architectures into the execution environment's model.
- **Supporting services**: Most projects require supporting services, such as databases, caches or language models. This serves to introduce the student to the hybrid cloud/open source ecosystem. It also builds skill in leveraging supporting services and more generally in discovering whether a particular problem has already been solved by someone else.
- **Releasing software**: This is both about the skill to package software into artifacts that can be deployed, but also about fitting releasing into the development workflow. The goal is to automate releasing as much as possible, removing any risk of inconsistency.
- **Observability**: This is the ability to understand what is happening in the execution environment, including logging, tracing, monitoring and alerting. The student will need to understand how to instrument their projects for observability and how to collect and interpret the data they provide.

## Infrastructure

The IaC space is dominated by Terraform (and its fork OpenTofu) which is also a good choice because of its three-way comparison model.

- **IaC coding**: In modern cloud environments, describing the resources needed to build a complete service is an integral part of development. The student should know how to configure their chosen execution environment and supporting services. Working with the infrastructure and coding in tandem fosters a devops mindset. We introduce IaC relatively early because it lends itself to reviewing.
- **Continuous delivery**: Once we have learned the basics of releasing and deploying software, we can automate the process. This builds on our understanding of process life cycles within the execution environment.I n later phases he student should confront zero downtime scenarios where services continue to operate during deployment, potentially requiring components of different versions to remain interoperable.
- **Multiple environments**: Most projects benefits from having dedicated development and/or testing environments. Some projects will be installed in multiple customer environments. The students will learn how to reason about configurability and how to manage diversity.

## Requirements

The student does not need to learn any formal methods for managing requirements or modelling user stories; simple text will do.

Agile methodologies tend to favor interaction and tight feedback cycles over working with requirements, but because the student have other tasks that take precedence (and stakeholders may also have other commitments), a slightly more formal requirements process allows the student to perform a full iteration even if it turns out that the need has changed in the meantime.

- **Requirements analysis**: The student needs to learn how to work with requirements, how to understand what they mean in software terms such as complexity and resource consumption. Ideally, the student gets to work with internal stakeholders that has some experience with expressing what they want. If not, the coach may need to facilitate the dialog.
- **Requirements collection**: Once the student has learned to analyze and refine requirements, they are ready to try to perform some collection of their own, by interviewing stakeholders to capture business needs and and helping them prioritize their requirements.

## Architecture

Students are not expected to tackle complex projects that require information or solution architecture work, but since they can be expected to work with and serverless technologies, a certain amount of software architecture skill is needed, such as when to use batch jobs and when to implement a long-running service?
