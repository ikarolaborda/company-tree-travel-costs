```php
<?php

class Travel {
    private $companyId;
    private $price;

    public function __construct($companyId, $price) {
        $this->companyId = $companyId;
        $this->price = $price;
    }

    public function getCompanyId() {
        return $this->companyId;
    }

    public function getPrice() {
        return $this->price;
    }
}

class Company {
    private $id;
    private $createdAt;
    private $name;
    private $parentId;
    private $cost = 0;
    private $children = [];

    public function __construct($id, $createdAt, $name, $parentId) {
        $this->id = $id;
        $this->createdAt = $createdAt;
        $this->name = $name;
        $this->parentId = $parentId;
    }

    public function getId() {
        return $this->id;
    }

    public function getCreatedAt() {
        return $this->createdAt;
    }

    public function getName() {
        return $this->name;
    }

    public function getParentId() {
        return $this->parentId;
    }  

    public function getCost() {
        return $this->cost;
    }

    public function getChildren() {
        return $this->children;
    }

    public function addChild(Company $child) {
        $this->children[] = $child;
    }

    public function calculateCost() {
        $this->cost = TravelRepository::getTravelCostByCompanyId($this->id);

        foreach ($this->children as $child) {
            $child->calculateCost();
            $this->cost += $child->getCost();
        }
    }

    public function toArray() {
        $array = [
            'id' => $this->id,
            'createdAt' => $this->createdAt,
            'name' => $this->name,
            'parentId' => $this->parentId,
            'cost' => $this->cost,
            'children' => []
        ];

        foreach ($this->children as $child) {
            $array['children'][] = $child->toArray();
        }

        return $array;
    }
}

class TravelRepository {
    private static $travelData = null;
    private static $cacheFile = 'travel_data.cache';
    private static $cacheExpiration = 3600;

    public static function fetchTravelData() {
        $cachePath = dirname(__FILE__) . '/' . self::$cacheFile;

        if (self::$travelData === null && file_exists($cachePath) && (time() - filemtime($cachePath)) < self::$cacheExpiration) {
            self::$travelData = json_decode(file_get_contents($cachePath), true);
        }

        if (self::$travelData === null) {
            $jsonData = file_get_contents('https://5f27781bf5d27e001612e057.mockapi.io/webprovise/travels');
            self::$travelData = json_decode($jsonData, true);
            file_put_contents($cachePath, json_encode(self::$travelData));
        }

        return self::$travelData;
    }

    public static function getTravelCostByCompanyId($companyId) {
        $travelData = self::fetchTravelData();
        $totalCost = 0;

        foreach ($travelData as $travel) {
            if ($travel['companyId'] == $companyId) {
                $totalCost += $travel['price'];
            }
        }

        return $totalCost;
    }
}

class CompanyRepository {
    private static $companyData = null;
    private static $cacheFile = 'company_data.cache';
    private static $cacheExpiration = 3600; // 1 hour in seconds

    public static function fetchCompanyData() {
        $cachePath = dirname(__FILE__) . '/' . self::$cacheFile;

        if (self::$companyData === null && file_exists($cachePath) && (time() - filemtime($cachePath)) < self::$cacheExpiration) {
            self::$companyData = json_decode(file_get_contents($cachePath), true);
        }

        if (self::$companyData === null) {
            $jsonData = file_get_contents('https://5f27781bf5d27e001612e057.mockapi.io/webprovise/companies');
            self::$companyData = json_decode($jsonData, true);
            file_put_contents($cachePath, json_encode(self::$companyData));
        }

        return self::$companyData;
    }

    public static function getCompaniesByParentId($parentId) {
        $companyData = self::fetchCompanyData();
        $matchingCompanies = [];

        foreach ($companyData as $company) {
            if ($company['parentId'] == $parentId) {
                $matchingCompanies[] = new Company(
                    $company['id'], 
                    $company['createdAt'],
                    $company['name'],
                    $company['parentId']
                );
            }
        }

        return $matchingCompanies;
    }

    public static function buildCompanyTree($companyData, $parentId = "0") {
        $tree = [];
        foreach ($companyData as $company) {
            if ($company['parentId'] == $parentId) {
                $child = new Company(
                    $company['id'], 
                    $company['createdAt'],
                    $company['name'],
                    $company['parentId']
                );
                $children = self::buildCompanyTree($companyData, $company['id']); // Get children as an array of Company objects
                foreach ($children as $childCompany) {
                    $child->addChild($childCompany); // Add each child individually
                }
                $tree[] = $child;
            }
        }
        return $tree;
    }
    
}


class TestScript {
    public function execute() {
        $start = microtime(true);

        $travelData = TravelRepository::fetchTravelData(); 
        $companyData = CompanyRepository::fetchCompanyData();
        $companies = CompanyRepository::buildCompanyTree($companyData);

        foreach ($companies as $company) {
            $company->calculateCost();
        }

        $companiesArray = [];
        foreach ($companies as $company) {
            $companiesArray[] = $company->toArray();
        }

        echo json_encode($companiesArray, JSON_PRETTY_PRINT);
        echo 'Total time: '.  (microtime(true) - $start);
    }
}


(new TestScript())->execute();

```

Outputs the results:
```json
[
    {
        "id": "uuid-1",
        "createdAt": "2021-02-26T00:55:36.632Z",
        "name": "Webprovise Corp",
        "parentId": "0",
        "cost": 52983,
        "children": [
            {
                "id": "uuid-2",
                "createdAt": "2021-02-25T10:35:32.978Z",
                "name": "Stamm LLC",
                "parentId": "uuid-1",
                "cost": 5199,
                "children": [
                    {
                        "id": "uuid-4",
                        "createdAt": "2021-02-25T06:11:47.519Z",
                        "name": "Price and Sons",
                        "parentId": "uuid-2",
                        "cost": 1340,
                        "children": []
                    },
                    {
                        "id": "uuid-7",
                        "createdAt": "2021-02-25T07:56:32.335Z",
                        "name": "Zieme - Mills",
                        "parentId": "uuid-2",
                        "cost": 1636,
                        "children": []
                    },
                    {
                        "id": "uuid-19",
                        "createdAt": "2021-02-25T21:06:18.777Z",
                        "name": "Schneider - Adams",
                        "parentId": "uuid-2",
                        "cost": 794,
                        "children": []
                    }
                ]
            },
            {
                "id": "uuid-3",
                "createdAt": "2021-02-25T15:16:30.887Z",
                "name": "Blanda, Langosh and Barton",
                "parentId": "uuid-1",
                "cost": 15713,
                "children": [
                    {
                        "id": "uuid-5",
                        "createdAt": "2021-02-25T13:35:57.923Z",
                        "name": "Hane - Windler",
                        "parentId": "uuid-3",
                        "cost": 1288,
                        "children": []
                    },
                    {
                        "id": "uuid-6",
                        "createdAt": "2021-02-26T01:41:06.479Z",
                        "name": "Vandervort - Bechtelar",
                        "parentId": "uuid-3",
                        "cost": 2512,
                        "children": []
                    },
                    {
                        "id": "uuid-9",
                        "createdAt": "2021-02-25T16:02:49.099Z",
                        "name": "Kuhic - Swift",
                        "parentId": "uuid-3",
                        "cost": 3086,
                        "children": []
                    },
                    {
                        "id": "uuid-17",
                        "createdAt": "2021-02-25T11:17:52.132Z",
                        "name": "Rohan, Mayer and Haley",
                        "parentId": "uuid-3",
                        "cost": 4072,
                        "children": []
                    },
                    {
                        "id": "uuid-20",
                        "createdAt": "2021-02-26T01:51:25.421Z",
                        "name": "Kunde, Armstrong and Hermann",
                        "parentId": "uuid-3",
                        "cost": 908,
                        "children": []
                    }
                ]
            },
            {
                "id": "uuid-8",
                "createdAt": "2021-02-25T23:47:57.596Z",
                "name": "Bartell - Mosciski",
                "parentId": "uuid-1",
                "cost": 28817,
                "children": [
                    {
                        "id": "uuid-10",
                        "createdAt": "2021-02-26T01:39:33.438Z",
                        "name": "Lockman Inc",
                        "parentId": "uuid-8",
                        "cost": 4288,
                        "children": []
                    },
                    {
                        "id": "uuid-11",
                        "createdAt": "2021-02-26T00:32:01.307Z",
                        "name": "Parker - Shanahan",
                        "parentId": "uuid-8",
                        "cost": 12236,
                        "children": [
                            {
                                "id": "uuid-12",
                                "createdAt": "2021-02-25T06:44:56.245Z",
                                "name": "Swaniawski Inc",
                                "parentId": "uuid-11",
                                "cost": 2110,
                                "children": []
                            },
                            {
                                "id": "uuid-14",
                                "createdAt": "2021-02-25T15:22:08.098Z",
                                "name": "Weimann, Runolfsson and Hand",
                                "parentId": "uuid-11",
                                "cost": 7254,
                                "children": []
                            }
                        ]
                    },
                    {
                        "id": "uuid-13",
                        "createdAt": "2021-02-25T20:45:53.518Z",
                        "name": "Balistreri - Bruen",
                        "parentId": "uuid-8",
                        "cost": 1686,
                        "children": []
                    },
                    {
                        "id": "uuid-15",
                        "createdAt": "2021-02-25T18:00:26.864Z",
                        "name": "Predovic and Sons",
                        "parentId": "uuid-8",
                        "cost": 4725,
                        "children": []
                    },
                    {
                        "id": "uuid-16",
                        "createdAt": "2021-02-26T01:50:50.354Z",
                        "name": "Weissnat - Murazik",
                        "parentId": "uuid-8",
                        "cost": 3277,
                        "children": []
                    }
                ]
            },
            {
                "id": "uuid-18",
                "createdAt": "2021-02-26T02:31:22.154Z",
                "name": "Walter, Schmidt and Osinski",
                "parentId": "uuid-1",
                "cost": 2033,
                "children": []
            }
        ]
    }
]Total time: 0.00061392784118652
```

# About this Approach
In this approach, I tried to make the most out of OOP, by using Repository Pattern. It demonstrates object-oriented programming principles, such as encapsulation and modularity, making the code organized and reusable. 
The use of caching for API data minimizes unnecessary external requests, improving performance. By calculating costs recursively and organizing companies into a tree structure, the script efficiently aggregates travel costs across a complex company hierarchy.
