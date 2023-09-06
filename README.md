# PulpoCon Workshop: Reliability Testing with Grafana k6

Useful links: Slides, [k6 docs](https://k6.io/docs/)

## 0. Before we start

### 0.1. Introduction
### 0.2. Requirements
- Docker & Docker Compose
- git
- GitHub/GitLab account
- Optional (but recommended): [k6](https://k6.io/docs/get-started/installation/)
  - You can run it inside Docker, but the experience is better if you install it locally!

### 0.3.  Run the local playground
First of all, clone the repository: 
```bash
git clone https://github.com/dgzlopes/pulpocon
```

Then, run the playground with:
```bash
cd pulpocon; docker-compose up -d
```

To verify that everything is working, go to http://localhost:3333 and click the big button. 

If you see a pizza recommendation, that's good!

Also, open http://localhost:3000 and verify that you can see a Grafana instance.
    
## 1. Foundations
### 1.1. What the heck is Grafana k6?
### 1.2. Run your first test

To run your first test, copy the following script:
```javascript
import http from "k6/http";

export default function () {
  let restrictions = {
    maxCaloriesPerSlice: 500,
    mustBeVegetarian: false,
    excludedIngredients: ["pepperoni"],
    excludedTools: ["knife"],
    maxNumberOfToppings: 6,
    minNumberOfToppings: 2
  }
  let res = http.post(`${BASE_URL}/api/pizza`, JSON.stringify(restrictions), {
    headers: {
      'Content-Type': 'application/json',
      'X-User-ID': 23423,
    },
  });
  console.log(`${res.json().pizza.name} (${res.json().pizza.ingredients.length} ingredients)`);
}
```

And paste it in a file named `example.js`.

Then, run it with:
```bash
# If you have k6 installed
k6 run example.js
# If you don't have k6 installed
docker run -i --network=pulpocon_default grafana/k6 run --quiet - <example.js
```

That's it! You have successfully run your first test.

Now, you got a lot of things in the output. Let's go through them. 

k6's output has three main sections:
- 1. Information about the test and its configuration.
  - Why: To quickly understand what's the test configuration and how it will run.
- 2. Runtime information (e.g. logs).
  - Why: To understand what's happening during the test.
- 3. Summary of the test.
  - Why: To understand how the test went.

Getting familiar with k6's output is important, as you will be using it a lot.

In the second section of the output, you should be able to see a log line with the pizza name and the number of ingredients. This happens once, because k6 has run your default function once. You can also see in the third section lots of metrics that k6 has generated and aggregated for you. These metrics are useful to understand how the test went (e.g. the number of requests, the number of errors, the response time, etc).

### 1.3. VUs and iterations

In k6 we have two main concepts: Virtual Users (VUs) and iterations. A VU is a virtual user. It's a thread that runs your script. It's the basic unit of execution in k6. An iteration is a single execution of your script. In the example above it's a single run of your default function.

You can think of it, like a for-loop. You can have many of those, and each one will run your script. The number of iterations is the number of times that your script will run. The number of VUs is the number of threads that will run your script.

When you run a k6 test, by default, it will run your script once, with a single VU. This is useful to verify that your script is working as expected. However, it's not very useful to understand how your system behaves under load. To do that, you need to increase the number of VUs and iterations.

Let's try it out! After the imports, add a configuration block to your script:
```javascript
import http from "k6/http";

export let options = {
  vus: 10,
};
```

Then, run it again:
```bash
# If you have k6 installed
k6 run example.js
# If you don't have k6 installed
docker run -i --network=pulpocon_default grafana/k6 run --quiet - <example.js
```

In the output, you should see many more log lines. That's because you have increased the number of VUs and we are getting more pizza recomendations. Lots of things change in the output when you increase the number of VUs, like: the values of the VUs and iterations metrics, the number of requests, the response time, etc.

#### 1.3.1. Think time

This is nice! But, we are hammering the service a bit too much. There is no wait/think time between iterations. Let's add a sleep to be a bit more gentle:
```javascript
// Add this import at the top of the file
import sleep from "k6";

// Add this line at the end of the default function
sleep(1);
```

Then, run the script again.

Now, we are getting a pizza recommendation every second. That's better! The number of iterations should be more predictable now. If you have 10VUs and I sleep for 1 second, you should get 10 iterations per second! 

If you want to have more RPS, you can increase the number of VUs. If you want to have less RPS, you can decrease the number of VUs.

#### 1.3.2. Stages

Now, let's try something more interesting. Let's say that we want to simulate a more realistic scenario. We want to simulate a ramp up of users, then a peak, and then a ramp down. We can do that with stages.

Replace the `options` block with the following:
```javascript
export let options = {
  stages: [
    { duration: "30s", target: 10 },
    { duration: "1m", target: 10 },
    { duration: "30s", target: 0 },
  ],
};
```

> TIP: You can always finish the test early, just by pressing CTRL+C. The output will still be generated!

Then, run the script again.

During the first 30 seconds k6 will ramp up from 0 to 10 VUs. Then, it will stay at 10 VUs for 1 minute. Finally, it will ramp down from 10 to 0 VUs in 30 seconds.

### 1.4. Checks

Another problem with our script is that we are not checking if the service is working as expected. We are just blindly sending requests and hoping for the best. We can do better than that! We can add checks to our script.

Checks are like assertions. They are a way to verify that the service is working as expected. 

Let's add a check to our script. First, we need to import the `check` function:
```javascript
import { check } from "k6";
```

Then, we can add a check to our script. You just need to add the following lines after the request:
```javascript
check(res, {
  "is status 200": (r) => r.status === 200,
});
```

Then, run the script again.

Now, you should see a new section in the output, with the results of the checks! You should see that all the checks passed. That's good!

If you want to see what happens when a check fails, you can change the check to:
```javascript
check(res, {
  "is status 200": (r) => r.status === 500,
});
```

### 1.5. Thresholds

Thresholds are the pass/fail criteria that you define for your test metrics. If the performance of the system under test (SUT) does not meet the conditions of your threshold, the test finishes with a failed status. That means, that k6 will exit with a non-zero exit code. You can leverage standard metrics that k6 generates, or custom metrics that you define in your script (we will see more about this later).

Let's add a threshold to our script. You can do that, by changing the `options` block to:
```javascript
export let options = {
  stages: [
    { duration: "30s", target: 10 },
    { duration: "1m", target: 10 },
    { duration: "30s", target: 0 },
  ],
  thresholds: {
    "http_req_duration": ["p(95)<5000"],
  },
};
```

Then, run the script again. You should see something nearby the metrics section, saying that the threshold passed.

This threshold is saying that 95% of the requests should be faster than 5 seconds. If that's not the case, the threshold fails.

To see what happens when a threshold fails, you can change the threshold to:
```javascript
export let options = {
  stages: [
    { duration: "30s", target: 10 },
    { duration: "1m", target: 10 },
    { duration: "30s", target: 0 },
  ],
  thresholds: {
    "http_req_duration": ["p(95)<10"],
  },
};
```

You can also inspect the status code of the test with:
```bash
# If you have k6 installed
echo $?

# If you don't have k6 installed
docker run -i --network=pulpocon_default grafana/k6 run --quiet - <example.js; echo $?
```

NOTE: Multiple ways of creating a threshold, and being able to finish the test early.

Then, run the script again.

### 1.6. Import data from a file

So far, we have been using hardcoded data. Let's change that!

### 1.7. Summary is nice but...
- Outputs: Let's see them! Graph a custom metric in Grafana.
Tip k6 run --out=experimental-prometheus-rw -e K6_PROMETHEUS_RW_STALE_MARKERS=true example.js

### 1.6. More stuff!

#### 1.6.1. CLI overrides 
#### 1.6.2. Environment variables
#### 1.6.3. Setup and teardown
#### 1.6.4. Custom metrics

## 2. Advanced

### 2.1. More than HTTP

### 2.2. Scenarios

### 2.3. Libraries

## 3. k6 + CI



