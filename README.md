# Mongodb with golang
mongo db connection with golang 


##### Mongo DB Connection 
```shell
package configs

import (
    "context"
    "fmt"
    "log"
    "time"
    "go.mongodb.org/mongo-driver/mongo"
    "go.mongodb.org/mongo-driver/mongo/options"
)

func ConnectDB() *mongo.Client  {
    client, err := mongo.NewClient(options.Client().ApplyURI("mongodb://localhost:27017"))
    if err != nil {
        log.Fatal(err)
    }

    ctx, _ := context.WithTimeout(context.Background(), 10*time.Second)
    err = client.Connect(ctx)
    if err != nil {
        log.Fatal(err)
    }

    //ping the database
    err = client.Ping(ctx, nil)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println("Connected to MongoDB")
    return client
}

//Client instance
var DB *mongo.Client = ConnectDB()

//getting database collections
func GetCollection(client *mongo.Client, collectionName string) *mongo.Collection {
    collection := client.Database("test_db").Collection(collectionName)
    return collection
}
```

##### Create user

```shell
package controllers

import (
    "context"
    "fiber-mongo-api/configs"
    "fiber-mongo-api/models"
    "fiber-mongo-api/responses"
    "net/http"
    "time"

    "github.com/go-playground/validator/v10"
    "github.com/gofiber/fiber/v2"
    "go.mongodb.org/mongo-driver/bson/primitive"
    "go.mongodb.org/mongo-driver/mongo"
)

var userCollection *mongo.Collection = configs.GetCollection(configs.DB, "users")
var validate = validator.New()

func CreateUser(c *fiber.Ctx) error {
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    var user models.User
    defer cancel()

    //validate the request body
    if err := c.BodyParser(&user); err != nil {
        return c.Status(http.StatusBadRequest).JSON(responses.UserResponse{Status: http.StatusBadRequest, Message: "error", Data: &fiber.Map{"data": err.Error()}})
    }

    //use the validator library to validate required fields
    if validationErr := validate.Struct(&user); validationErr != nil {
        return c.Status(http.StatusBadRequest).JSON(responses.UserResponse{Status: http.StatusBadRequest, Message: "error", Data: &fiber.Map{"data": validationErr.Error()}})
    }

    newUser := models.User{
        Id:       primitive.NewObjectID(),
        Name:     user.Name,
        Location: user.Location,
        Title:    user.Title,
    }

    result, err := userCollection.InsertOne(ctx, newUser)
    if err != nil {
        return c.Status(http.StatusInternalServerError).JSON(responses.UserResponse{Status: http.StatusInternalServerError, Message: "error", Data: &fiber.Map{"data": err.Error()}})
    }

    return c.Status(http.StatusCreated).JSON(responses.UserResponse{Status: http.StatusCreated, Message: "success", Data: &fiber.Map{"data": result}})
}

func GetAUser(c *fiber.Ctx) error {
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    userId := c.Params("userId")
    var user models.User
    defer cancel()

    objId, _ := primitive.ObjectIDFromHex(userId)

    err := userCollection.FindOne(ctx, bson.M{"id": objId}).Decode(&user)
    if err != nil {
        return c.Status(http.StatusInternalServerError).JSON(responses.UserResponse{Status: http.StatusInternalServerError, Message: "error", Data: &fiber.Map{"data": err.Error()}})
    }

    return c.Status(http.StatusOK).JSON(responses.UserResponse{Status: http.StatusOK, Message: "success", Data: &fiber.Map{"data": user}})
}

func EditAUser(c *fiber.Ctx) error {
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    userId := c.Params("userId")
    var user models.User
    defer cancel()

    objId, _ := primitive.ObjectIDFromHex(userId)

    //validate the request body
    if err := c.BodyParser(&user); err != nil {
        return c.Status(http.StatusBadRequest).JSON(responses.UserResponse{Status: http.StatusBadRequest, Message: "error", Data: &fiber.Map{"data": err.Error()}})
    }

    //use the validator library to validate required fields
    if validationErr := validate.Struct(&user); validationErr != nil {
        return c.Status(http.StatusBadRequest).JSON(responses.UserResponse{Status: http.StatusBadRequest, Message: "error", Data: &fiber.Map{"data": validationErr.Error()}})
    }

    update := bson.M{"name": user.Name, "location": user.Location, "title": user.Title}

    result, err := userCollection.UpdateOne(ctx, bson.M{"id": objId}, bson.M{"$set": update})
    if err != nil {
        return c.Status(http.StatusInternalServerError).JSON(responses.UserResponse{Status: http.StatusInternalServerError, Message: "error", Data: &fiber.Map{"data": err.Error()}})
    }

    //get updated user details
    var updatedUser models.User
    if result.MatchedCount == 1 {
        err := userCollection.FindOne(ctx, bson.M{"id": objId}).Decode(&updatedUser)
        if err != nil {
            return c.Status(http.StatusInternalServerError).JSON(responses.UserResponse{Status: http.StatusInternalServerError, Message: "error", Data: &fiber.Map{"data": err.Error()}})
        }
    }

    return c.Status(http.StatusOK).JSON(responses.UserResponse{Status: http.StatusOK, Message: "success", Data: &fiber.Map{"data": updatedUser}})
}

func DeleteAUser(c *fiber.Ctx) error {
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    userId := c.Params("userId")
    defer cancel()

    objId, _ := primitive.ObjectIDFromHex(userId)

    result, err := userCollection.DeleteOne(ctx, bson.M{"id": objId})
    if err != nil {
        return c.Status(http.StatusInternalServerError).JSON(responses.UserResponse{Status: http.StatusInternalServerError, Message: "error", Data: &fiber.Map{"data": err.Error()}})
    }

    if result.DeletedCount < 1 {
        return c.Status(http.StatusNotFound).JSON(
            responses.UserResponse{Status: http.StatusNotFound, Message: "error", Data: &fiber.Map{"data": "User with specified ID not found!"}},
        )
    }

    return c.Status(http.StatusOK).JSON(
        responses.UserResponse{Status: http.StatusOK, Message: "success", Data: &fiber.Map{"data": "User successfully deleted!"}},
    )
}

func GetAllUsers(c *fiber.Ctx) error {
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    var users []models.User
    defer cancel()

    results, err := userCollection.Find(ctx, bson.M{})

    if err != nil {
        return c.Status(http.StatusInternalServerError).JSON(responses.UserResponse{Status: http.StatusInternalServerError, Message: "error", Data: &fiber.Map{"data": err.Error()}})
    }

    //reading from the db in an optimal way
    defer results.Close(ctx)
    for results.Next(ctx) {
        var singleUser models.User
        if err = results.Decode(&singleUser); err != nil {
            return c.Status(http.StatusInternalServerError).JSON(responses.UserResponse{Status: http.StatusInternalServerError, Message: "error", Data: &fiber.Map{"data": err.Error()}})
        }

        users = append(users, singleUser)
    }

    return c.Status(http.StatusOK).JSON(
        responses.UserResponse{Status: http.StatusOK, Message: "success", Data: &fiber.Map{"data": users}},
    )
}

```
##### Routers 
```shell
package routes

import (
    "fiber-mongo-api/controllers"
    "github.com/gofiber/fiber/v2"
)

func UserRoute(app *fiber.App) {
    app.Post("/user", controllers.CreateUser)
    app.Get("/user/:userId", controllers.GetAUser)
    app.Put("/user/:userId", controllers.EditAUser)
    app.Delete("/user/:userId", controllers.DeleteAUser)
    app.Get("/users", controllers.GetAllUsers)
}
```

##### Models
```shell
package models

import "go.mongodb.org/mongo-driver/bson/primitive"

type User struct {
    Id       primitive.ObjectID `json:"id,omitempty"`
    Name     string             `json:"name,omitempty" validate:"required"`
    Location string             `json:"location,omitempty" validate:"required"`
    Title    string             `json:"title,omitempty" validate:"required"`
}
```


