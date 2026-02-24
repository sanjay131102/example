void MapPaintWidget::setThemeInternal(const QString& themeName, Pos radarPos)
{

    qDebug() << Q_FUNC_INFO;
    if (!layout) {
        layout = new QVBoxLayout(this);
    }

    auto getNearbyTilePaths_subtiles = [=](double radarLat, double radarLon, const QString& themeName) -> QList<QString>
    {
        QList<QString> existingTiles;

        constexpr double radiusKm = 40.0;
        constexpr double latStep = 0.25;
        constexpr double lonStep = 0.25;

        // Convert to radians
        double radLat = radarLat * M_PI / 180.0;
        double kmPerDegLat = 111.0;
        double kmPerDegLon = 111.0 * std::cos(radLat);

        // Bounding box of search area in degrees
        double radiusLatDeg = radiusKm / kmPerDegLat;
        double radiusLonDeg = radiusKm / kmPerDegLon;

        double minLat = radarLat - radiusLatDeg;
        double maxLat = radarLat + radiusLatDeg;
        double minLon = radarLon - radiusLonDeg;
        double maxLon = radarLon + radiusLonDeg;

        int baseLatMin = static_cast<int>(std::floor(minLat));
        int baseLatMax = static_cast<int>(std::floor(maxLat));
        int baseLonMin = static_cast<int>(std::floor(minLon));
        int baseLonMax = static_cast<int>(std::floor(maxLon));

        // Helper to test if subtile bbox intersects circle of radiusKm around radar
        auto subtileIntersectsCircle = [&](double minLatB, double maxLatB, double minLonB, double maxLonB) -> bool
        {
            // Convert subtile bounds to km relative to radar (origin at 0,0)
            double minLatKm = (minLatB - radarLat) * kmPerDegLat;
            double maxLatKm = (maxLatB - radarLat) * kmPerDegLat;
            double minLonKm = (minLonB - radarLon) * kmPerDegLon;
            double maxLonKm = (maxLonB - radarLon) * kmPerDegLon;

            // Find closest point on bbox to origin (0,0)
            double closestX = 0.0;
            if (0 < minLonKm)
                closestX = minLonKm;
            else if (0 > maxLonKm)
                closestX = maxLonKm;

            double closestY = 0.0;
            if (0 < minLatKm)
                closestY = minLatKm;
            else if (0 > maxLatKm)
                closestY = maxLatKm;

            double distSq = closestX * closestX + closestY * closestY;
            return distSq <= radiusKm * radiusKm;
        };

        for (int tileLat = baseLatMin; tileLat <= baseLatMax; ++tileLat) {
            for (int tileLon = baseLonMin; tileLon <= baseLonMax; ++tileLon) {
                QString latPrefix = (tileLat >= 0) ? "n" : "s";
                QString lonPrefix = (tileLon >= 0) ? "e" : "w";
                QString folderName = latPrefix + QString::number(std::abs(tileLat)) +
                                     lonPrefix + QString::number(std::abs(tileLon));
                QString tileFolder = QDir::currentPath() + "/../themes/" + themeName + "/" + folderName;

                for (int row = 0; row < 4; ++row) {
                    for (int col = 0; col < 4; ++col) {
                        // Subtile bounding box in degrees
                        double subtileMinLat = tileLat + 1.0 - (row + 1) * latStep;  // bottom edge
                        double subtileMaxLat = tileLat + 1.0 - row * latStep;        // top edge
                        double subtileMinLon = tileLon + col * lonStep;              // left edge
                        double subtileMaxLon = tileLon + (col + 1) * lonStep;        // right edge

                        if (subtileIntersectsCircle(subtileMinLat, subtileMaxLat, subtileMinLon, subtileMaxLon)) {
                            int subtileIndex = row * 4 + col;
                            QString subtilePath = tileFolder + "/" + QString("%1.tpkx").arg(subtileIndex, 3, 10, QLatin1Char('0'));
                            bool exists = QFile::exists(subtilePath);
                            qDebug() << "Candidate:" << subtilePath << (exists ? "[Exists]" : "[Missing]");
                            if (exists) {
                                existingTiles.append(subtilePath);
                            }
                        }
                    }
                }
            }
        }

        // qDebug() << "Total .tpkx files to load:" << existingTiles.size();
        return existingTiles;
    };
    auto getNearbyTilePaths = [=](double radarLat, double radarLon, const QString& themeName) -> QList<QString>
    {
        QList<QString> existingTiles;

        constexpr double radiusKm = 40.0;

        // Convert to radians
        double radLat = radarLat * M_PI / 180.0;
        double kmPerDegLat = 111.0;
        double kmPerDegLon = 111.0 * std::cos(radLat);

        // Bounding box of search area in degrees
        double radiusLatDeg = radiusKm / kmPerDegLat;
        double radiusLonDeg = radiusKm / kmPerDegLon;

        double minLat = radarLat - radiusLatDeg;
        double maxLat = radarLat + radiusLatDeg;
        double minLon = radarLon - radiusLonDeg;
        double maxLon = radarLon + radiusLonDeg;

        int latMinDeg = static_cast<int>(std::floor(minLat));
        int latMaxDeg = static_cast<int>(std::floor(maxLat));
        int lonMinDeg = static_cast<int>(std::floor(minLon));
        int lonMaxDeg = static_cast<int>(std::floor(maxLon));

        auto tileIntersectsCircle = [&](double tileLat, double tileLon) -> bool
        {
            double minLatKm = (tileLat - radarLat) * kmPerDegLat;
            double maxLatKm = (tileLat + 1.0 - radarLat) * kmPerDegLat;
            double minLonKm = (tileLon - radarLon) * kmPerDegLon;
            double maxLonKm = (tileLon + 1.0 - radarLon) * kmPerDegLon;

            double closestX = 0.0;
            if (0 < minLonKm) closestX = minLonKm;
            else if (0 > maxLonKm) closestX = maxLonKm;

            double closestY = 0.0;
            if (0 < minLatKm) closestY = minLatKm;
            else if (0 > maxLatKm) closestY = maxLatKm;

            double distSq = closestX * closestX + closestY * closestY;
            return distSq <= radiusKm * radiusKm;
        };

        for (int tileLat = latMinDeg; tileLat <= latMaxDeg; ++tileLat) {
            for (int tileLon = lonMinDeg; tileLon <= lonMaxDeg; ++tileLon) {

                if (!tileIntersectsCircle(tileLat, tileLon))
                    continue;

                QString latPrefix = (tileLat >= 0) ? "n" : "s";
                QString lonPrefix = (tileLon >= 0) ? "e" : "w";
                QString tileName = latPrefix + QString::number(std::abs(tileLat)).rightJustified(2, '0') +
                                   lonPrefix + QString::number(std::abs(tileLon)).rightJustified(3, '0');

                QString filePath = QDir::currentPath() + "/../3d_data/" + tileName + ".tif";

                if (QFile::exists(filePath)) {
                    existingTiles.append(filePath);
                    qDebug() << "Found:" << filePath;
                } else {
                    qDebug() << "Missing:" << filePath;
                }
            }
        }

        // qDebug() << "Total .tiff files to load:" << existingTiles.size();
        return existingTiles;
    };
    QList<Esri::ArcGISRuntime::GraphicsOverlay*> savedOverlays;
    QList<Esri::ArcGISRuntime::GraphicsOverlay*> savedOverlays3d;
    QList<Esri::ArcGISRuntime::Layer*> savedLayers;


    if(MapProjection==0)//2d
    {
        if(m_sceneView)
        {
            // Copy overlays before deleting mapView
            auto oldOverlays = m_sceneView->graphicsOverlays();
            for (int i = 0; i < oldOverlays->size(); ++i)
            {
                Esri::ArcGISRuntime::GraphicsOverlay* ov = oldOverlays->at(i);
                savedOverlays3d.append(ov);
            }
            qDebug() << "Overlay size m_sceneView" << savedOverlays3d.size();
            layout->removeWidget(m_sceneView);
            delete m_sceneView;
            m_sceneView=nullptr;
        }

        if (m_mapView)
        {
            // --- Save graphics overlays ---
            auto oldOverlays = m_mapView->graphicsOverlays();
            for (int i = 0; i < oldOverlays->size(); ++i)
            {
                Esri::ArcGISRuntime::GraphicsOverlay* ov = oldOverlays->at(i);
                savedOverlays.append(ov);
            }
            qDebug() << "Overlay size m_mapView" << savedOverlays.size();

            // --- Save operational layers ---
            if (m_mapView->map())
            {
                auto oldLayers = m_mapView->map()->operationalLayers();
                for (int i = 0; i < oldLayers->size(); ++i)
                {
                    Esri::ArcGISRuntime::Layer* layer = oldLayers->at(i);
                    savedLayers.append(layer);
                }
                qDebug() << "Operational layer size m_mapView" << savedLayers.size();
            }

            layout->removeWidget(m_mapView);
            delete m_mapView;
            m_mapView = nullptr;
        }


        if (m_sceneMaintenanceTimer)
        {
            m_sceneMaintenanceTimer->stop();
            m_sceneMaintenanceTimer->deleteLater();
            m_sceneMaintenanceTimer = nullptr;
        }

        // Fetch tile paths within 40km radius (intersecting subtiles)
        QList<QString> tilePaths = getNearbyTilePaths_subtiles(radarPos.getLatY(), radarPos.getLonX(), themeName);

        // Load all nearby tiles as layers
        QList<Esri::ArcGISRuntime::Layer*> tileLayers;

        if (tilePaths.isEmpty()) {
            qWarning() << "No .tpkx files found within 40km of radar.";
            QMessageBox::warning(this,
                                 tr("Missing Map Data"),
                                 tr("No map tiles (.tpkx) were found around the radar location.\n"
                                    "Loading only India basemap."));

            // return;
        }else{
            for (const QString& path : tilePaths) {
                qDebug() << "Loading TPKX from:" << path;
                auto* tileCache = new Esri::ArcGISRuntime::TileCache(path, this);
                auto* tiledLayer = new Esri::ArcGISRuntime::ArcGISTiledLayer(tileCache, this);
                tileLayers.append(tiledLayer);
            }
        }


        // Create a basemap from the first tile layer and append others
        Esri::ArcGISRuntime::Basemap* basemap = new Esri::ArcGISRuntime::Basemap(this);

        QString relativePath1 = QCoreApplication::applicationDirPath() + "/../india/india.tpkx";

        QString modelPath1 = QDir::cleanPath(QDir(QCoreApplication::applicationDirPath()).filePath(relativePath1));

        Esri::ArcGISRuntime::TileCache* tileCache1 = new Esri::ArcGISRuntime::TileCache(modelPath1, this);
        Esri::ArcGISRuntime::ArcGISTiledLayer* tiledLayer0 = new Esri::ArcGISRuntime::ArcGISTiledLayer(tileCache1, this);
        basemap->baseLayers()->append(tiledLayer0); // Add first TPKX layer
        if(!tileLayers.isEmpty())
        {
            // qDebug()<<"index india "<<basemap->baseLayers()->indexOf(tiledLayer0);
            basemap->baseLayers()->append(tileLayers.first());
            // qDebug()<<"index india "<<basemap->baseLayers()->indexOf(tileLayers.first());

            for (int i = 1; i < tileLayers.size(); ++i) {
                basemap->baseLayers()->append(tileLayers[i]);
                // qDebug()<<"index 16 files "<<basemap->baseLayers()->indexOf(tileLayers[i]);
            }
        }

        m_mapView = new Esri::ArcGISRuntime::MapGraphicsView(this);
        m_mapView->setMagnifierEnabled(true);

        // m_mapView->setMouseTracking(true);  // Important for MouseMove without pressing button
        // m_mapView->installEventFilter(this);

        auto* map = new Esri::ArcGISRuntime::Map(basemap, this);
        m_mapView->setMap(map);

        // code fo grid
        // // Create a latitude-longitude grid
        grid = new Esri::ArcGISRuntime::LatitudeLongitudeGrid(this);

        // Configure grid properties
        grid->setLabelPosition(Esri::ArcGISRuntime::GridLabelPosition::Geographic);
        grid->setLabelFormat(Esri::ArcGISRuntime::LatitudeLongitudeGridLabelFormat::DegreesMinutesSeconds);
        grid->setVisible(true);

        // Apply grid to the MapView based on the ui option
        // m_mapView->setGrid(grid);

        // Ensure we only create once
        //layer creation
        // Restore overlays
        auto overlays = m_mapView->graphicsOverlays();
        qDebug() <<savedOverlays.size()<<"size of saved overlay";
        if (!savedOverlays.isEmpty())
        {
            for (auto* ov : savedOverlays)
            {
                overlays->append(ov);
            }
            qDebug() << "Restored" << savedOverlays.size() << "overlays into new mapView.";
            savedOverlays.clear();
        }
        else
        {
            // Create fresh overlays if none were saved
            auto* radarOverlay   = new Esri::ArcGISRuntime::GraphicsOverlay(this);
            auto* trailOverlay   = new Esri::ArcGISRuntime::GraphicsOverlay(this);
            auto* vehicleOverlay = new Esri::ArcGISRuntime::GraphicsOverlay(this);
            auto* MeasureOverlay = new Esri::ArcGISRuntime::GraphicsOverlay(this);
            auto* rangering = new Esri::ArcGISRuntime::GraphicsOverlay(this);

            overlays->append(radarOverlay);
            overlays->append(trailOverlay);
            overlays->append(vehicleOverlay);
            overlays->append(MeasureOverlay);
            overlays->append(rangering);

            qDebug() << "Created fresh overlays.";
        }

        // Restore operational layers
        if (!savedLayers.isEmpty())
        {
            for (auto* layer : savedLayers)
                m_mapView->map()->operationalLayers()->append(layer);

            loadKml(kmlFilePaths.first(),false);
            qDebug() << "yes  op restore";

        }else
        {
            qDebug() << "no op restore";

        }


        //Layer debugging
        if (overlays)
        {
            qDebug() << "Total overlays:" << overlays->size();
            for (int i = 0; i < overlays->size(); ++i)
            {
                Esri::ArcGISRuntime::GraphicsOverlay* ov = overlays->at(i);

                qDebug().nospace()
                    << "Overlay[" << i << "] "
                    << ", graphics count=" << ov->graphics()->size()
                    << ", visible=" << (ov->isVisible() ? "true" : "false");
            }
        }
        else
        {
            qDebug() << "No overlays list available!";
        }

        emit mapViewLoaded();
        emit disable3DTools();
        paintLayer->updateLayers();

        // m_graphicsOverlayListModel = m_mapView->graphicsOverlays();
        // m_graphicsOverlayListModel->append(paintLayer->context.painter);

        // Initial viewpoint around radar (100 km radius approx)

        if(!currentPos.isValid())
        {
        Esri::ArcGISRuntime::Viewpoint viewpoint(
            Esri::ArcGISRuntime::Point(radarPos.getLonX(), radarPos.getLatY(), Esri::ArcGISRuntime::SpatialReference::wgs84()),
            100000);
        map->setInitialViewpoint(viewpoint);
        }
        else
        {
            Esri::ArcGISRuntime::Viewpoint viewpoint(Esri::ArcGISRuntime::Point(currentPos.getLonX(), currentPos.getLatY(), Esri::ArcGISRuntime::SpatialReference::wgs84()),100000);
            map->setInitialViewpoint(viewpoint);
        }
        // Layout
        layout->setContentsMargins(0, 0, 0, 0);
        layout->addWidget(m_mapView);
        setLayout(layout);

    }
    // else if (MapProjection == 1) // 3D mode
    // {

    //     if (m_mapView)
    //     {
    //         // --- Save graphics overlays ---
    //         auto oldOverlays = m_mapView->graphicsOverlays();
    //         for (int i = 0; i < oldOverlays->size(); ++i)
    //         {
    //             Esri::ArcGISRuntime::GraphicsOverlay* ov = oldOverlays->at(i);
    //             savedOverlays.append(ov);
    //         }
    //         qDebug() << "Overlay size m_mapView" << savedOverlays.size();

    //         // --- Save operational layers ---
    //         if (m_mapView->map())
    //         {
    //             auto oldLayers = m_mapView->map()->operationalLayers();
    //             for (int i = 0; i < oldLayers->size(); ++i)
    //             {
    //                 Esri::ArcGISRuntime::Layer* layer = oldLayers->at(i);
    //                 savedLayers.append(layer);
    //             }
    //             qDebug() << "Operational layer size m_mapView" << savedLayers.size();
    //         }

    //         layout->removeWidget(m_mapView);
    //         delete m_mapView;
    //         m_mapView = nullptr;
    //     }

    //     if(m_sceneView)
    //     {
    //         // Copy overlays before deleting mapView
    //         auto oldOverlays = m_sceneView->graphicsOverlays();
    //         for (int i = 0; i < oldOverlays->size(); ++i)
    //         {
    //             Esri::ArcGISRuntime::GraphicsOverlay* ov = oldOverlays->at(i);
    //             savedOverlays3d.append(ov);
    //         }
    //         qDebug() << "Overlay size m_sceneView" << savedOverlays3d.size();
    //         layout->removeWidget(m_sceneView);
    //         delete m_sceneView;
    //         m_sceneView=nullptr;
    //     }

    //     QStringList elevationFiles;
    //     // Create the scene (3D map)
    //     m_scene = new Esri::ArcGISRuntime::Scene(this);
    //     // ✅ Create Basemap from RasterLayer and TiledLayer
    //     Esri::ArcGISRuntime::Basemap* rasterBasemap = new Esri::ArcGISRuntime::Basemap(this);
    //     // ✅ Stage 1: Load .tiff full tiles setp 1
    //     QList<QString> tilePaths = getNearbyTilePaths(radarPos.getLatY(), radarPos.getLonX(), themeName);
    //     for (const QString& path : tilePaths) {
    //         QString fullPath = QDir::cleanPath(QDir(QCoreApplication::applicationDirPath()).filePath(path));
    //         if (fullPath.endsWith(".tif", Qt::CaseInsensitive) || fullPath.endsWith(".tiff", Qt::CaseInsensitive)) {
    //             Esri::ArcGISRuntime::Raster* raster = new Esri::ArcGISRuntime::Raster(fullPath, this);
    //             Esri::ArcGISRuntime::RasterLayer* rasterLayer = new Esri::ArcGISRuntime::RasterLayer(raster, this);
    //             rasterBasemap->baseLayers()->append(rasterLayer);
    //             qDebug() << "Loaded TIFF Raster:" << fullPath;
    //             elevationFiles << fullPath; // Add multiple elevation files

    //         }
    //     }
    //     // ✅ Load Multiple Elevation TIFFs

    //     Esri::ArcGISRuntime::RasterElevationSource* elevationSource = new Esri::ArcGISRuntime::RasterElevationSource(elevationFiles, this);

    //     // Debug: Check if elevation source loads
    //     connect(elevationSource, &Esri::ArcGISRuntime::RasterElevationSource::loadStatusChanged, this, [](Esri::ArcGISRuntime::LoadStatus status) {
    //         qDebug() << "Elevation source load status:" << static_cast<int>(status);
    //     });

    //     elevationSource->load();  // Ensure the files load

    //     // ✅ Stage 2: Create a Surface and add the Elevation Sources
    //     Esri::ArcGISRuntime::Surface* elevationSurface = new Esri::ArcGISRuntime::Surface(this);
    //     elevationSurface->elevationSources()->append(elevationSource);

    //     QString relativePath3 = "../3d_data/wsiearth.tif";  // Move up from build directory
    //     QString modelPath3 = QDir::cleanPath(QDir(QCoreApplication::applicationDirPath()).filePath(relativePath3));
    //     // ✅ Load Raster for Basemap
    //     Esri::ArcGISRuntime::Raster* raster = new Esri::ArcGISRuntime::Raster(modelPath3, this);
    //     Esri::ArcGISRuntime::RasterLayer* rasterLayer = new Esri::ArcGISRuntime::RasterLayer(raster, this);
    //     rasterBasemap->baseLayers()->append(rasterLayer);

    //     // ✅ Stage 3: Load .tpkx subtiles
    //     QList<QString> subtilePaths = getNearbyTilePaths_subtiles(radarPos.getLatY(), radarPos.getLonX(), themeName);
    //     for (const QString& path : subtilePaths) {
    //         QString fullPath = QDir::cleanPath(QDir(QCoreApplication::applicationDirPath()).filePath(path));
    //         if (fullPath.endsWith(".tpkx", Qt::CaseInsensitive)) {
    //             Esri::ArcGISRuntime::TileCache* tileCache = new Esri::ArcGISRuntime::TileCache(fullPath, this);
    //             Esri::ArcGISRuntime::ArcGISTiledLayer* tiledLayer = new Esri::ArcGISRuntime::ArcGISTiledLayer(tileCache, this);
    //             rasterBasemap->baseLayers()->append(tiledLayer);
    //             qDebug() << "Loaded TPKX Subtile:" << fullPath;
    //         }
    //     }

    //     grid = new Esri::ArcGISRuntime::LatitudeLongitudeGrid(this);

    //     // Configure grid properties
    //     grid->setLabelPosition(Esri::ArcGISRuntime::GridLabelPosition::Geographic);
    //     grid->setLabelFormat(Esri::ArcGISRuntime::LatitudeLongitudeGridLabelFormat::DegreesMinutesSeconds);
    //     grid->setVisible(true);

    //     // ✅ Set this surface as the base for the scene
    //     m_scene->setBaseSurface(elevationSurface);
    //     m_scene->setBasemap(rasterBasemap);

    //     m_sceneView = new Esri::ArcGISRuntime::SceneGraphicsView(this);

    //     // Finally set the scene on your SceneView
    //     m_sceneView->setArcGISScene(m_scene);

    //     int wkid = m_sceneView->arcGISScene()->spatialReference().wkid();
    //     qDebug() << "SceneView SpatialReference WKID:" << wkid;

    //     // Restore overlays
    //     auto overlays = m_sceneView->graphicsOverlays();
    //     if (!savedOverlays3d.isEmpty())
    //     {
    //         for (auto* ov : savedOverlays3d)
    //         {
    //             overlays->append(ov);
    //         }
    //         qDebug() << "Restored" << savedOverlays3d.size() << "overlays into new mapView.";
    //         savedOverlays3d.clear();
    //     }
    //     else
    //     {
    //     // Create a graphics overlay for 3D objects
    //     Esri::ArcGISRuntime::GraphicsOverlay* overlay3D =
    //         new Esri::ArcGISRuntime::GraphicsOverlay(this);
    //     Esri::ArcGISRuntime::GraphicsOverlay* overlay3D_aircraft =
    //         new Esri::ArcGISRuntime::GraphicsOverlay(this);

    //     // Set placement: Absolute means using real-world Z values
    //     overlay3D->setSceneProperties(
    //         Esri::ArcGISRuntime::LayerSceneProperties(
    //             Esri::ArcGISRuntime::SurfacePlacement::Absolute
    //             )
    //         );
    //     overlay3D_aircraft->setSceneProperties(
    //         Esri::ArcGISRuntime::LayerSceneProperties(
    //             Esri::ArcGISRuntime::SurfacePlacement::Absolute
    //             )
    //         );
    //     // Append overlay to the SceneView
    //     m_sceneView->graphicsOverlays()->insert(0, overlay3D);
    //     m_sceneView->graphicsOverlays()->insert(1, overlay3D_aircraft);
    //     }
    //     paintLayer->updateLayers();
    //     // m_sceneView->graphicsOverlays()->append(paintLayer->context.painter);

    //     layout->setContentsMargins(0, 0, 0, 0);
    //     layout->addWidget(m_sceneView);
    //     setLayout(layout);
    //     emit mapViewLoaded();
    //     emit disable2DTools();
    //     qDebug() <<currentPos<<"current pos";
    //     if(!currentPos.isValid())
    //     {
    //         Esri::ArcGISRuntime::Camera camera(radarPos.getLatY(), radarPos.getLonX(), 8000.0, 0.0, 0, 0.0);
    //         m_sceneView->setViewpointCamera(camera, 0.0); // instant move, no animation        }

    //     }else
    //     {
    //         Esri::ArcGISRuntime::Camera camera(currentPos.getLatY(), currentPos.getLonX(), 8000.0, 0.0, 0, 0.0);
    //         m_sceneView->setViewpointCamera(camera, 0.0); // instant move, no animation        }
    //     }

    //     m_viewpointUpdateTimer = new QTimer(this);
    //     m_viewpointUpdateTimer->setInterval(200); // 200 ms debounce
    //     m_viewpointUpdateTimer->setSingleShot(true);

    // }
    else if (MapProjection == 1) // 3D mode testting only
    {
        if (m_mapView)
        {
            auto oldOverlays = m_mapView->graphicsOverlays();
            for (int i = 0; i < oldOverlays->size(); ++i)
                savedOverlays.append(oldOverlays->at(i));
            if (m_mapView->map())
            {
                auto oldLayers = m_mapView->map()->operationalLayers();
                for (int i = 0; i < oldLayers->size(); ++i)
                    savedLayers.append(oldLayers->at(i));
            }
            layout->removeWidget(m_mapView);
            delete m_mapView;
            m_mapView = nullptr;
        }

        if (m_sceneView)
        {
            auto oldOverlays = m_sceneView->graphicsOverlays();
            for (int i = 0; i < oldOverlays->size(); ++i)
                savedOverlays3d.append(oldOverlays->at(i));
            layout->removeWidget(m_sceneView);
            delete m_sceneView;
            m_sceneView = nullptr;
        }

        m_scene = new Esri::ArcGISRuntime::Scene(this);
        Esri::ArcGISRuntime::Basemap* rasterBasemap = new Esri::ArcGISRuntime::Basemap(this);

        // ---- ONLY wsiearth.tif ----
        QString modelPath3 = QDir::cleanPath(
            QDir(QCoreApplication::applicationDirPath())
                .filePath("../3d_data/wsiearth.tif"));
        Esri::ArcGISRuntime::Raster* raster = new Esri::ArcGISRuntime::Raster(modelPath3, this);
        Esri::ArcGISRuntime::RasterLayer* rasterLayer = new Esri::ArcGISRuntime::RasterLayer(raster, this);
        rasterBasemap->baseLayers()->append(rasterLayer);

        // Flat surface — no elevation
        Esri::ArcGISRuntime::Surface* elevationSurface = new Esri::ArcGISRuntime::Surface(this);
        m_scene->setBaseSurface(elevationSurface);
        m_scene->setBasemap(rasterBasemap);

        grid = new Esri::ArcGISRuntime::LatitudeLongitudeGrid(this);
        grid->setLabelPosition(Esri::ArcGISRuntime::GridLabelPosition::Geographic);
        grid->setLabelFormat(Esri::ArcGISRuntime::LatitudeLongitudeGridLabelFormat::DegreesMinutesSeconds);
        grid->setVisible(true);

        m_sceneView = new Esri::ArcGISRuntime::SceneGraphicsView(this);
        m_sceneView->setArcGISScene(m_scene);

        // Always fresh overlays — no restore
        auto* overlay3D = new Esri::ArcGISRuntime::GraphicsOverlay(this);
        auto* overlay3D_aircraft = new Esri::ArcGISRuntime::GraphicsOverlay(this);

        overlay3D->setSceneProperties(
            Esri::ArcGISRuntime::LayerSceneProperties(
                Esri::ArcGISRuntime::SurfacePlacement::Absolute));
        overlay3D_aircraft->setSceneProperties(
            Esri::ArcGISRuntime::LayerSceneProperties(
                Esri::ArcGISRuntime::SurfacePlacement::Absolute));

        m_sceneView->graphicsOverlays()->insert(0, overlay3D);
        m_sceneView->graphicsOverlays()->insert(1, overlay3D_aircraft);

        paintLayer->updateLayers();

        layout->setContentsMargins(0, 0, 0, 0);
        layout->addWidget(m_sceneView);
        setLayout(layout);
        emit mapViewLoaded();
        emit disable2DTools();

        // Camera to Chennai for test
        Esri::ArcGISRuntime::Camera camera(13.0827, 80.2707, 8000.0, 0.0, 0, 0.0);
        m_sceneView->setViewpointCamera(camera, 0.0);

        m_viewpointUpdateTimer = new QTimer(this);
        m_viewpointUpdateTimer->setInterval(200);
        m_viewpointUpdateTimer->setSingleShot(true);

        // ---- START CHENNAI TEST DRONE (no UDP) ----
        QTimer::singleShot(2000, this, [this]() {

            auto* sceneView = getMapSceneView();
            if (!sceneView || sceneView->graphicsOverlays()->size() <= 1) return;

            auto* overlay = sceneView->graphicsOverlays()->at(1);
            if (!overlay) return;

            if (m_wgs84.wkid() != 4326)
                m_wgs84 = Esri::ArcGISRuntime::SpatialReference::wgs84();

            if (!m_droneSymbol)
            {
                QString modelPath = QDir::cleanPath(
                    QDir(QCoreApplication::applicationDirPath())
                        .filePath("../3d_data/Bristol.dae"));

                if (!QFile::exists(modelPath))
                {
                    qWarning() << "Bristol.dae not found";
                    return;
                }

                m_droneSymbol = new Esri::ArcGISRuntime::ModelSceneSymbol(
                    QUrl::fromLocalFile(modelPath), 50.0);

                // m_droneSymbol->setAnchorPosition(
                //     Esri::ArcGISRuntime::SceneSymbolAnchorPosition::Bottom);
            }

            double centerLat = 13.0827;
            double centerLon = 80.2707;
            double altitude  = 2000.0;
            double radius    = 0.05;

            auto* graphic = new Esri::ArcGISRuntime::Graphic(
                Esri::ArcGISRuntime::Point(
                    centerLon + radius, centerLat, altitude, m_wgs84),
                m_droneSymbol);

            graphic->attributes()->insertAttribute("ANGLE", 0.0);
            overlay->graphics()->append(graphic);

            if (!overlay->renderer())
            {
                auto* renderer = new Esri::ArcGISRuntime::SimpleRenderer(m_droneSymbol);
                Esri::ArcGISRuntime::RendererSceneProperties props;
                props.setHeadingExpression("[ANGLE]");
                renderer->setSceneProperties(props);
                overlay->setRenderer(renderer);
            }

            double* angle = new double(0.0);
            QTimer* rotTimer = new QTimer(this);
            rotTimer->setInterval(250);

            QObject::connect(rotTimer, &QTimer::timeout, this, [=]() {
                *angle += 2.0;
                if (*angle >= 360.0) *angle = 0.0;

                double rad = *angle * M_PI / 180.0;
                double newLon = centerLon + radius * std::cos(rad);
                double newLat = centerLat + radius * std::sin(rad);

                graphic->setGeometry(
                    Esri::ArcGISRuntime::Point(newLon, newLat, altitude, m_wgs84));

                graphic->attributes()->replaceAttribute("ANGLE", *angle);
            });

            rotTimer->start();
            qDebug() << "[Test] Chennai drone rotating — watching for stutter";
        });
    }





}
