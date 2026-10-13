---
title: "Fragmenting a god controller, for a tab pane in JavaFX"    
date: 2026-10-11
summary: "Set the controller to implement a Coordinator that  handles between tab communication, and used fx:include in each tab"  
---
## The Problem?
- TabPaneController was handling all communication between each tab and became cluttered and hard to read. (704 lines)
- This meant every tab had access to mainManager directly as it was calle dthrough tabPaneControllers initialise.

## The Proposed Solution?
- Break up the functionality of each tab pane into its own controller and use parse in seperate fxmls on each tab of the tab-pane
- Use a TabPaneCoordinator interface that handles switching between the tabs.


### TabPaneController (God Class)
- **Errors highlighted with !!!**
```java
public void initialise(MainManager mainManager, MapController mapController, ScreenNavigator navigator) {
        this.mainManager = mainManager;
        this.mapController = mapController;
        this.navigator = navigator;
    }

/**
 * Lengthy tabs init -> Moved to fxml file
   **/
public void initialiseTabs() { particularBarsScrollpane.visibleProperty().bind(Bindings.isNotEmpty(selectedParticularBars());particularBarsScrollpane.managedProperty().bind(particularBarsVBox.visibleProperty());
	disableSaveButton();
	showRouteTab();
	setupParameters();
	initialiseSaveRouteName();
	initialiseRouteDescription();
	initialiseSpinner();
	emptyRouteDetails();
	menuExpanded = false;
	toggleMenu();
	barFeaturesCheckComboBox.getItems().addAll(FeatureType.values());
	barThemesCheckComboBox.getItems().addAll(Theme.values());
	setApproxDistanceFieldNumeric();
	setNumBarsSpinnerNumeric();
}

/**
 * "Helper method to check if the inputted name in..."
!!! Logic should be moved to service
 */
private boolean nameTaken(String newRoute) {
	if (newRoute.isEmpty()) return false;
	String trimmed =  newRoute.trim();
	for (String saved : mainManager.getSavedRouteNames()) {
		if (saved != null && saved.trim().equals(trimmed)) {
			return true;
		}
	}
	return false;
}

/**
 * "Initilises the saveRouteName listener error feedback.."
   !!! Repeated use of error feedback should be extracted to utils
 */
private void initialiseSaveRouteName() {
	routeNameField.setTextFormatter(new TextFormatter<>(change -> {
		if (change.getControlNewText().length() <= MAX_ROUTE_NAME) {
			routeNameErrorLabel.setText("");
			routeNameField.getStyleClass().remove(errorInFieldCssClass);
			return change;
		}
		showRedBorderError(routeNameField);
		showRedTextError(routeNameErrorLabel, String.format("Route name cannot exceed %d characters", MAX_ROUTE_NAME));
		return null;
	}));
	routeNameField.textProperty().addListener((observable, oldValue, newRouteName) -> {
		if (nameTaken(newRouteName) && !newRouteName.isEmpty()) {
			showRedBorderError(routeNameField);
			showRedTextError(routeNameErrorLabel, "Route name already exists");
		} else {
			routeNameErrorLabel.setText("");
			routeNameField.getStyleClass().remove(errorInFieldCssClass);
		}
	}
	);
}

private void initialiseRouteDescription() {
	routeDescriptionField.setTextFormatter(new TextFormatter<>(change -> {
		if (change.getControlNewText().length() <= MAX_DESCRIPTION) {
			routeNameErrorLabel.setText("");
			routeDescriptionField.getStyleClass().remove(errorInFieldCssClass);
			return change;
		}
		showRedBorderError(routeDescriptionField);
		showRedTextError(routeNameErrorLabel, String.format("Description cannot exceed %d characters", MAX_DESCRIPTION));
		return null;
	}));
}
/**
More error Handling utils
.
.
.
/**
/**
 * Displays and populates the route info grid
 * Switching between tabs can be moved outside
 */
public void displayRouteDetails(Route route) {
	// Clear route info cell
	routeDetailsVBox.setVisible(true);
	routeDetailsVBox.setManaged(true);
	mainManager.setCurrentRoute(route);
	routeDetailsTabVBox.getChildren().remove(noRouteGrid);
	showDetailsTab();
	routeBarsVBox.getChildren().clear();
	loadBarCards(route, routeBarsVBox);
}

public void displayRouteName(Route route) {
	routeNameField.setText(route.getName());
	routeDescriptionField.setText(route.getDescription());

	routeNameErrorLabel.setText("");
	routeNameField.getStyleClass().remove(errorInFieldCssClass);
}

/**
 * Adds bar cards to given container
!!!! GUI shouldnt have direct access to core application informaton
 */
public void loadBarCards(Route route, VBox destination) {
	if (route != null) {
		walkingDistanceLabel.setText(String.format("%.1f", route.getTotalDistanceKm()));
		estTimeLabel.setText(route.getFormattedTime());
		int barNum = 1;
		for (Bar bar : route.getBars()) {
			BarCardWidget.addRouteBarCard(bar, destination, barNum, route.getNumStops(),
					b -> mainManager.venueInfoWidget.show(b));
			barNum++;
		}
	}
}

/**
!!! Repeated formating to be moved to utils 
**/
private void setApproxDistanceFieldNumeric() {
	approxDistanceField.setTextFormatter(new TextFormatter<>(
			new DoubleStringConverter(),
			null,
			c -> c.getControlNewText().matches("\\d*(\\.\\d?)?") ? c : null)
	);
}
private void setNumBarsSpinnerNumeric() {
	numBarsSpinner.getEditor().setTextFormatter(new TextFormatter<>(
			new IntegerStringConverter(),
			null,
			c -> c.getControlNewText().matches("[2-9]?") ? c : null)
	);
}

private void newParticularBarCard(Bar bar) {
	BarCardWidget barCardWidget = new BarCardWidget();
	barCardWidget.addParticularBarCard(bar, particularBarsVBox, selectedParticularBars::remove, b -> mainManager.venueInfoWidget.show(b));
}

/**
Tab swapping logic
.
.
.
**/

/**
 * Grabs values from GUI and passes them
 * through to RouteBuilder to create a new route
   !!! Logic being directly called in GUI is a no-no!
   This function also has direct access to mainmanager containing
   database access and user details
 */
@FXML
public void generateRoute() {
	clearRedBorderError();
	clearRedTextError();

	String startBar = startBarField.getText();
	String endBar = endBarField.getText();
	Double approximateDistanceInKm = !Objects.equals(approxDistanceField.getText(), "") ? Double.parseDouble(approxDistanceField.getText()) : null;
	int numberOfBars = (numBarsSpinner.getValue() == null) ? 0 : numBarsSpinner.getValue();
	List<Theme> themes = barThemesCheckComboBox.getCheckModel().getCheckedItems();
	List<FeatureType> features = barFeaturesCheckComboBox.getCheckModel().getCheckedItems();
	List<Bar> particularBars = new ArrayList<>(selectedParticularBars);

	try {

		RouteParameterInputs routeParameterInputs = new RouteParameterInputs(startBar, endBar, numberOfBars, approximateDistanceInKm, themes, features, particularBars);

		RouteService.generateRoutes(mainManager, routeParameterInputs, 5);
		Route generatedRoute = mainManager.getGeneratedRoute(0);
		List<Bar> chosenBars = generatedRoute.getBars();
		displayRouteDetails(generatedRoute);
		mapController.displayRoute(chosenBars);
		clearNameAndDescription();
		showDetailsTab();
		addRegenButton();
	} catch (InvalidParametersException e) {
		InvalidParametersReason reason = e.getReason();
		switch (reason) {
			case START_BAR_EMPTY -> {
				showRedBorderError(startBarField);
				showRedTextError(startBarErrorLabel, "Must select a start bar");
			}
			case START_BAR_DOESNT_EXIST -> {
				showRedBorderError(startBarField);
				showRedTextError(startBarErrorLabel, "Bar does not exist");
			}
			case END_BAR_DOESNT_EXIST -> {
				showRedBorderError(endBarField);
				showRedTextError(endBarErrorLabel, "Bar does not exist");
			}
			case TOO_MANY_PARTICULAR_BARS -> {
				showRedBorderError(particularBarsSearchField);
				showRedTextError(particularBarsErrorLabel, "Too many particular bars selected");
			}
			case TOO_MANY_THEMES -> {
				showRedBorderError(barThemesCheckComboBox);
				showRedTextError(themesErrorLabel, "Too many themes selected");
			}
		}
	} catch (NoRouteGeneratedException e) {
		showPopupError("Could not build a route", e.getMessage());
	}
}


/**
 * Calls method to take map snapshot and launches the export window
 * !!! Repeated Alert logic deserves seperate function
 */
@FXML
public void callSaveRoute() {
	User currentUser = mainManager.getCurrentUser();
	Route route = mainManager.getCurrentRoute();
	updateRouteDetails(route);
	if (route.getName() == null || route.getName().isEmpty()) {
		route.setName("GENERATED ROUTE");
	}t
	if (mainManager.routeDAO.saveRoute(route, currentUser.getId())) {
		mainManager.setUserRoutes();
		loadSavedRoutes(mainManager.getUserRoutes(), mainManager.getUserId());
		Alert alert = new Alert(Alert.AlertType.INFORMATION);
		alert.setTitle("Save Route");
		alert.setHeaderText(null);
		alert.setContentText("Route Successfully Saved!");
		alert.showAndWait();
		showSavedTab();
	} else {
		Alert alert = new Alert(Alert.AlertType.INFORMATION);
		alert.setTitle("Save Route");
		alert.setHeaderText(null);
		alert.setContentText("Route Unable To Be Saved");
		alert.showAndWait();
	}
}

/**
 * Deletes the current user's account
   !!! Take logic out 
 */
@FXML
public void deleteAccount() {
	Alert alert = new Alert(Alert.AlertType.CONFIRMATION);
	alert.setTitle("Delete Account");
	alert.setHeaderText("Are you sure you want to delete this account?");
	alert.setContentText("This action cannot be undone");
	Optional<ButtonType> result = alert.showAndWait();
	if (result.isPresent() && result.get() == ButtonType.OK) {
		mainManager.deleteUser();
		navigator.goToSetup(false);
	}
}

/**
Extra code...
**/

/**
!!! Repeated error feedback should be moved to seperate UTIL 
**/
private void showPopupError(String header, String message) {
	Alert alert = new Alert(Alert.AlertType.ERROR);
	alert.setTitle("Route not generated");
	alert.setHeaderText(header);
	alert.setContentText(message);
	alert.initOwner(generateRouteButton.getScene().getWindow());
	alert.showAndWait();
}

private void showRedBorderError(Node node) {
	node.getStyleClass().add(errorInFieldCssClass);
}

private void showRedTextError(Label label, String errorMessage) {
	label.setText(errorMessage);
}

private void clearRedBorderError() {
	startBarField.getStyleClass().remove(errorInFieldCssClass);
	endBarField.getStyleClass().remove(errorInFieldCssClass);
	particularBarsVBox.getStyleClass().remove(errorInFieldCssClass);
	barThemesCheckComboBox.getStyleClass().remove(errorInFieldCssClass);
}

private void clearRedTextError() {
	startBarErrorLabel.setText("");
	endBarErrorLabel.setText("");
	particularBarsErrorLabel.setText("");
	themesErrorLabel.setText("");
}

}
```
## Task 1: Create Utility Classes 
  
To remove repeat code, we seperated the styling, alert pop-ups into a util's with static methods

#### FieldError (replacing "showPopupError", etc...)
```java
public final class FieldErrorUtil {  

    public static void show(Node field, Label label, String message) {  
        if (!field.getStyleClass().contains(ERROR_IN_FIELD_CSS_CLASS)) {  
            field.getStyleClass().add(ERROR_IN_FIELD_CSS_CLASS);  
        }  
        label.setText(message);  
    }  
  
    public static void clear(Node field, Label label) {  
        field.getStyleClass().remove(ERROR_IN_FIELD_CSS_CLASS);  
        label.setText("");  
    }  

    public static void clearBorders(Node... fields) {  
        for (Node field : fields) {  
            field.getStyleClass().remove(ERROR_IN_FIELD_CSS_CLASS);  
        }  
    }  
  
	public static void clearTexts(Label... labels) {  
        for (Label label : labels) {  
            label.setText("");  
        }  
    }  
}
```

#### AlertBox (replacing "showPopupError", etc...)
  ```java
public final class DialogUtil {  
  
    public static boolean confirm(String title, String header, String content) {  
        Alert alert = new Alert(Alert.AlertType.CONFIRMATION);  
        alert.setTitle(title);  
        alert.setHeaderText(header);  
        alert.setContentText(content);  
        applyStyle(alert);  
        Optional<ButtonType> result = alert.showAndWait();  
        return result.isPresent() && result.get() == ButtonType.OK;  
    }  

    public static void info(String title, String content) {  
        Alert alert = new Alert(Alert.AlertType.INFORMATION);  
        alert.setTitle(title);  
        alert.setHeaderText(null);  
        alert.setContentText(content);  
        applyStyle(alert);  
        alert.showAndWait();  
    }  
  
	public static void error(Window own, String t, String h, String c) {  
        Alert alert = new Alert(Alert.AlertType.ERROR);  
        alert.setTitle(t);  
        alert.setHeaderText(h);  
        alert.setContentText(c);  
        applyStyle(alert);  
        if (owner != null) {  
            alert.initOwner(owner);  
        }  
        alert.showAndWait();  
    }  

    private static void applyStyle(Alert alert) {    
	alert.getDialogPane().getStylesheets().
	add(Objects.requireNonNull(DialogUtil.class.
	getResource(STYLESHEET)).toExternalForm());  
        alert.getDialogPane().getStyleClass().add(DIALOG_CSS_CLASS);  
        alert.setGraphic(null);  
    }  
}
  ```

## Task 2: Set up the Coordinator to switch between tabs

As all the services required access to information stored in mainManager, I knew that each of them had to be init like this at some point. 
```java
routeBuilderService = new RouteBuilderService(mainManager);  
savedRoutesService = new SavedRoutesService(mainManager);  
communityRoutesService = new CommunityRoutesService(mainManager);  
accountService = new AccountService(mainManager);
```

However I didnt want each controller to be directly interacting with the mainManager or all of TabPanControllers account functions. So instead I used an interface that contained the tab switching methods that TabPaneController implemented meaning the other tabs could switch, without ever holding references to each other. 

```java
public interface TabPaneCoordinator {  
  
    void showRouteBuilderTab();  
  
    void showRouteDetailsTab();  
  
    void showSavedTab();  
  
    void showCommunityTab();  
  
    void returnToSetup();  
  
    void showGeneratedRoute(Route route);  

    void openRoute(Route route);  
  
    void refreshSavedRoutes();  
  
    void resetRouteView();  
  
    void regenerateRoute();  
}
```

Which was then overriden in TabPaneController e.g...
```java
@Override  
@FXML  
public void showDetailsTab() {  
    showTab(viewTab);  
    detailsTabButton.setSelected(true);  
}
```

## Task 3: Split up God Controller

I started by seperating the controller into 4 other controllers RouteBuilderTab, RouteDetailsTab, SavedRoutesTab, and CommunityRoutesTab. This went smoothly as their was little entanglement between the tabs, except for loading routes, (Which proved to be a tougher fix than originally thought.) After spliting up by functionality, I tried to take as much logic out of the classes as possible and place them inside each service. 

#### SavedRoutesTabController (example)
```java
public class SavedRoutesTabController {  
  
    @FXML  
    private VBox savedRouteVBox;  
    @FXML  
    private GridPane savedGuestGrid;  
    @FXML  
    private Button signInSavedButton;  
    @FXML  
    private Label savedTabLabel;  
  
    private SavedRoutesService savedRoutesService;  
    private TabPaneCoordinator coordinator;  
  
	public void initialise(SavedRoutesService savedRoutesService,
	TabPaneCoordinator coordinator) {  
        this.savedRoutesService = savedRoutesService;  
        this.coordinator = coordinator;  
    }  

	public void load() {  
        load(savedRoutesService.getSavedRoutes());  
    }  
  
	public void load(List<Route> savedRoutes) {  
        savedRouteVBox.getChildren().clear();  
        if (savedRoutes.isEmpty()) {  
            noSavedRoutes();  
        }  
        for (Route route : savedRoutes) {  
            RouteCardWidget.addSavedRouteCard(  
                    route,  
                    savedRouteVBox,  
                    coordinator::openRoute,  
                    this::confirmAndDelete,  
                    savedRoutesService::publishRoute);  
        }  
    }  
	  private void noSavedRoutes() {  
        savedRouteVBox.getChildren().add(savedGuestGrid);  
        signInSavedButton.setText("New Route");  
        signInSavedButton.setOnAction(e -> coordinator.showRouteTab());  
        savedTabLabel.setText("You have no saved routes");  
    }  
  
    public void returnToSetup() {  
        coordinator.returnToSetup();  
    }  
  
    private void confirmAndDelete(Route route) {  
        if (DialogUtil.confirm("Delete Saved Route", 
	        "Are you sure you want to delete this route?",  
	        "This action cannot be undone")) {  
            savedRoutesService.deleteRoute(route);  
            load();  
            coordinator.resetRouteView();  
        }  
    }  
}
```

#### SavedRoutesService (example)
```java
package seng202.team5.services;  
  
import seng202.team5.gui.util.DialogUtil;  
import seng202.team5.models.Route;  
  
import java.util.List;  

public class SavedRoutesService {  
    public static final String DEFAULT_ROUTE_NAME = "GENERATED ROUTE";  
  
    public enum SaveResult {  
        SAVED,  
        NAME_TAKEN,  
        FAILED  
    }  
  
    private final MainManager mainManager;  
  


    public SavedRoutesService(MainManager mainManager) {  
        this.mainManager = mainManager;  
    }  
  

    public boolean isNameTaken(String name) {  
        if (name == null || name.isBlank()) {  
            return false;  
        }  
        String trimmed = name.trim();  
        for (String saved : mainManager.getSavedRouteNames()) {  
            if (saved != null && saved.trim().equals(trimmed)) {  
                return true;  
            }  
        }  
        return false;  
    }  
  


     public SaveResult saveRoute(Route route, String name, String description) {  
        String finalName = (name == null || name.isBlank()) ? 
        DEFAULT_ROUTE_NAME : name;  
        if (isNameTaken(finalName)) {  
            return SaveResult.NAME_TAKEN;  
        }  
        route.setName(finalName);  
        route.setDesc(description);  
  
        boolean saved = mainManager.routeDAO.saveRoute(route,
        mainManager.getCurrentUser().getId());  
        if (!saved) {  
            return SaveResult.FAILED;  
        }  
        mainManager.setUserRoutes();  
        return SaveResult.SAVED;  
    }  
  


    public List<Route> getSavedRoutes() {  
        return mainManager.getUserRoutes();  
    }  
  

    public void deleteRoute(Route route) {  
        mainManager.routeDAO.deleteSavedRoute(route.getRouteID());  
        mainManager.setUserRoutes();  
    }  
  
	public void publishRoute(Route route) {  
        mainManager.routeDAO.publishCommunityRoute(route,
        mainManager.getUserId());  
        DialogUtil.info("Route Published", "Route Successfully Published!");  
    }  
}
```
## Conclusion: Great Success?
This refactor took away from TabPaneControllers god-hood, but unfortunately was unable to properly address MainManager(User Information + DB Access) being completely public for the GUI to interact with. While this issue does stem primarily from the dependence on mainManager to hold important  information, the team and I thought we would be able to address with the controllers refactor.

## What would I do differently?
Now I know how utils and widgets can help tidy up repeat code, establish which parts of the app should be widgets would be my first step. If we had started with this, we would have spent less time on making entire cards by copying and pasting, to just calling its constructor. In terms of god-controllers, this was definitely a step in the right direction. Utilising an interface was not something we had been taught yet but I saw surprising success. 
